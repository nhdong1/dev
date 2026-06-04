# 09 — Monitoring & Observability — Giám Sát & Quan Sát Toàn Diện

> CloudWatch — Nền Tảng Giám Sát Trung Tâm của AWS — kết hợp với X-Ray (Theo Dõi Phân Tán) tạo nên hệ thống observability (quan sát) đầy đủ cho mọi AWS Compute workload

## 📚 Mục Lục

1. [Tại Sao Monitoring Quan Trọng](#tại-sao-monitoring-quan-trọng)
2. [Kiến Trúc Observability Trên AWS](#kiến-trúc-observability-trên-aws)
3. [Ba Trụ Cột Observability](#ba-trụ-cột-observability)
4. [Tổng Quan Các File](#tổng-quan-các-file)
5. [Chiến Lược Monitoring Theo Dịch Vụ](#chiến-lược-monitoring-theo-dịch-vụ)
6. [SLI, SLO, SLA — Đo Lường Chất Lượng Dịch Vụ](#sli-slo-sla)
7. [Luồng Cảnh Báo — Alert Flow](#luồng-cảnh-báo)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Monitoring Quan Trọng

Monitoring không chỉ là "biết hệ thống đang làm gì" — mà là **phát hiện vấn đề trước khi người dùng bị ảnh hưởng**, hiểu xu hướng để lập kế hoạch capacity, và có đủ dữ liệu để debug khi sự cố xảy ra.

### Bốn Mục Tiêu Cốt Lõi

```
┌─────────────────────────────────────────────────────────┐
│  1. DETECT (Phát Hiện)                                  │
│     → Biết ngay khi có vấn đề, không đợi user báo cáo  │
│                                                         │
│  2. DIAGNOSE (Chẩn Đoán)                                │
│     → Tìm root cause (nguyên nhân gốc rễ) nhanh chóng  │
│                                                         │
│  3. OPTIMIZE (Tối Ưu)                                   │
│     → Dùng dữ liệu thực tế để cải thiện hiệu năng      │
│                                                         │
│  4. PLAN (Lập Kế Hoạch)                                 │
│     → Dự báo capacity, tránh bất ngờ về chi phí        │
└─────────────────────────────────────────────────────────┘
```

### Hậu Quả Của Thiếu Monitoring

| Tình Huống                           | Hậu Quả                                     |
| ------------------------------------ | ------------------------------------------- |
| Không có alarm                       | Sự cố kéo dài nhiều giờ, user mới báo cáo  |
| Không có logs đủ chi tiết            | Mất nhiều giờ debug, không tìm được nguyên nhân |
| Không có distributed tracing         | Không biết service nào gây latency cao      |
| Không theo dõi chi phí              | Shock cuối tháng khi nhận hóa đơn AWS       |
| Không có dashboards                  | Không thể đánh giá tình trạng hệ thống nhanh |

---

## Kiến Trúc Observability Trên AWS

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS OBSERVABILITY STACK                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DATA SOURCES (Nguồn Dữ Liệu)                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │   EC2    │ │  Lambda  │ │   ECS    │ │   EKS    │              │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘              │
│       │             │             │             │                   │
│  ─────┴─────────────┴─────────────┴─────────────┴──────────────    │
│                                                                     │
│  COLLECTION (Thu Thập)                                              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  CloudWatch     │  │   X-Ray Daemon   │  │  CloudWatch      │  │
│  │  Agent          │  │   / ADOT         │  │  Logs Agent      │  │
│  └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                    │                      │            │
│  STORAGE & PROCESSING (Lưu Trữ & Xử Lý)               │            │
│  ┌─────────────────────────────────────────────────┐  │            │
│  │              AMAZON CLOUDWATCH                  │  │            │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │  │            │
│  │  │ Metrics  │ │  Logs    │ │    Alarms      │  │  │            │
│  │  │(Chỉ Số) │ │(Nhật Ký) │ │  (Cảnh Báo)   │  │  │            │
│  │  └──────────┘ └──────────┘ └────────────────┘  │  │            │
│  └─────────────────────────────────────────────────┘  │            │
│                                                        │            │
│  ┌─────────────────────────────────────────────────┐  │            │
│  │              AWS X-RAY                          │◄─┘            │
│  │  Service Map (Bản Đồ Dịch Vụ) + Trace Analysis │               │
│  └─────────────────────────────────────────────────┘               │
│                                                                     │
│  ACTION (Hành Động)                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │   SNS    │ │  Auto    │ │ Systems  │ │ Lambda   │              │
│  │ Notify   │ │ Scaling  │ │ Manager  │ │ Remediate│              │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Ba Trụ Cột Observability

Observability (Khả Năng Quan Sát Hệ Thống) được xây dựng trên **3 trụ cột**:

### 1. Metrics — Chỉ Số Đo Lường

**Định nghĩa:** Số liệu định lượng theo thời gian (time-series data)

```
Ví dụ:
- EC2 CPUUtilization = 78% lúc 14:30:00
- Lambda Duration = 245ms (trung bình 1 phút)
- ECS RunningTasksCount = 12 tasks
```

**Đặc điểm:** Nhẹ, lưu trữ lâu dài, dễ vẽ đồ thị và tạo alarm

**AWS Service:** Amazon CloudWatch Metrics

### 2. Logs — Nhật Ký Sự Kiện

**Định nghĩa:** Bản ghi văn bản của các sự kiện xảy ra trong hệ thống

```
Ví dụ:
2026-05-15T14:30:01Z ERROR [OrderService] Failed to process order #12345: 
  Connection timeout to payment-service:8080 after 5000ms
  at com.example.OrderService.processPayment(OrderService.java:156)
```

**Đặc điểm:** Chi tiết, giúp debug, nhưng tốn storage hơn metrics

**AWS Service:** Amazon CloudWatch Logs

### 3. Traces — Vết Theo Dõi Phân Tán

**Định nghĩa:** Theo dõi hành trình của một request xuyên qua nhiều services

```
Request ID: abc-123
  ├── API Gateway (12ms)
  ├── Lambda [order-handler] (245ms)
  │   ├── DynamoDB GetItem (8ms)
  │   ├── SQS SendMessage (15ms)
  │   └── HTTP → payment-service (210ms) ← BOTTLENECK!
  └── Total: 267ms
```

**Đặc điểm:** Giúp tìm bottleneck trong hệ thống phân tán, thấy dependency giữa services

**AWS Service:** AWS X-Ray

---

## Tổng Quan Các File

### 📄 [1-cloudwatch-metrics.md](./1-cloudwatch-metrics.md)

**CloudWatch Metrics — Chỉ Số Giám Sát**

- Default metrics (chỉ số mặc định) cho EC2, Lambda, ECS, EKS
- Custom metrics (chỉ số tùy chỉnh) — cách publish từ ứng dụng
- Metric dimensions (chiều dữ liệu) và namespaces (không gian tên)
- High-resolution metrics (chỉ số độ phân giải cao) — 1 giây
- CloudWatch Agent cho metrics nâng cao (RAM, disk, process)

### 📄 [2-cloudwatch-logs.md](./2-cloudwatch-logs.md)

**CloudWatch Logs — Quản Lý Nhật Ký**

- Log Groups (Nhóm Nhật Ký) và Log Streams (Luồng Nhật Ký)
- CloudWatch Logs Insights — ngôn ngữ query phân tích log
- Metric Filters — trích xuất metrics từ log entries
- Retention policies (chính sách lưu giữ) và cost management
- Log aggregation từ EC2, Lambda, ECS, EKS

### 📄 [3-cloudwatch-alarms.md](./3-cloudwatch-alarms.md)

**CloudWatch Alarms — Hệ Thống Cảnh Báo**

- Alarm states: OK, ALARM, INSUFFICIENT_DATA
- Threshold-based alarms (cảnh báo ngưỡng) và anomaly detection
- Composite Alarms (Cảnh Báo Tổng Hợp) — kết hợp nhiều điều kiện
- Actions: SNS notification, Auto Scaling, EC2 recovery
- Alarm math — tính toán trên nhiều metrics

### 📄 [4-xray-tracing.md](./4-xray-tracing.md)

**AWS X-Ray — Distributed Tracing**

- Trace (Vết), Segment (Đoạn), Subsegment (Đoạn Con) — cấu trúc dữ liệu
- X-Ray Service Map (Bản Đồ Dịch Vụ) — visualize dependencies
- Sampling rules (quy tắc lấy mẫu) — kiểm soát chi phí
- Integration với Lambda, ECS, EC2, API Gateway
- X-Ray Groups và Insights — phân tích nâng cao

### 📄 [5-observability-checklist.md](./5-observability-checklist.md)

**Observability Checklist — Danh Mục Kiểm Tra**

- SLI (Service Level Indicator — Chỉ Số Mức Dịch Vụ)
- SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ)
- SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ)
- Dashboard design best practices
- Runbook (Sổ Tay Vận Hành) templates
- On-call rotation và escalation policies

---

## Chiến Lược Monitoring Theo Dịch Vụ

### EC2 — Monitoring Máy Chủ Ảo

```
MỨC ĐỘ 1 — DEFAULT (Miễn phí, 5 phút/lần):
  ✓ CPUUtilization       → Alert nếu > 80% trong 15 phút
  ✓ NetworkIn/Out        → Phát hiện traffic bất thường
  ✓ DiskReadOps/WriteOps → Monitor I/O bottleneck
  ✓ StatusCheckFailed    → Alert ngay, trigger auto-recovery

MỨC ĐỘ 2 — DETAILED (Tốn phí, 1 phút/lần):
  ✓ Granular CPU data    → Cần cho Auto Scaling nhạy hơn
  ✓ Per-instance metrics → Quan trọng trong ASG

MỨC ĐỘ 3 — CloudWatch Agent (Custom, cài đặt thêm):
  ✓ MemoryUtilization    → RAM — không có trong default!
  ✓ DiskUsage            → Dung lượng ổ đĩa
  ✓ Process metrics      → Số lượng process đang chạy
```

### Lambda — Monitoring Serverless

```
KPI QUAN TRỌNG NHẤT:
  ✓ Errors             → Error rate (tỷ lệ lỗi) = Errors / Invocations
  ✓ Duration           → P50/P95/P99 latency — P99 quan trọng hơn P50!
  ✓ Throttles          → Đang bị giới hạn concurrency
  ✓ ConcurrentExecs    → Gần chạm limit chưa?
  ✓ IteratorAge        → Nếu dùng Kinesis/DynamoDB Streams, lag bao nhiêu?

ĐỌC LOG Lambda: luôn có correlation ID để link trace → log
```

### ECS — Monitoring Container

```
CLUSTER LEVEL:
  ✓ CPUReservation / MemoryReservation → Cluster có dư capacity?
  ✓ CPUUtilization / MemoryUtilization → Đang dùng bao nhiêu?

SERVICE LEVEL:
  ✓ RunningTaskCount  → Đủ task đang chạy không?
  ✓ PendingTaskCount  → Task đang pending — thiếu capacity?

CONTAINER LEVEL (cần Container Insights):
  ✓ container_cpu_utilized
  ✓ container_memory_utilized
  ✓ container_network_*
```

### EKS — Monitoring Kubernetes

```
CONTROL PLANE:
  ✓ API Server latency  → cluster_request_rate, cluster_failed_request_rate
  ✓ etcd metrics        → etcd_object_counts, etcd_request_duration

NODE LEVEL:
  ✓ node_cpu_utilization
  ✓ node_memory_utilization
  ✓ node_disk_io_utilization
  ✓ node_network_total_bytes

POD LEVEL:
  ✓ pod_cpu_utilized / pod_memory_utilized
  ✓ pod_number_of_container_restarts → restarts cao = vấn đề
```

---

## SLI, SLO, SLA

### Định Nghĩa và Mối Quan Hệ

```
SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ)
  ↑ Cam kết với khách hàng / hợp đồng pháp lý
  ↑ Ví dụ: "99.9% availability, bồi thường nếu vi phạm"

SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ)
  ↑ Mục tiêu nội bộ team — NGHIÊM NGẶT HƠN SLA
  ↑ Ví dụ: "99.95% availability" (buffer so với SLA 99.9%)

SLI (Service Level Indicator — Chỉ Số Mức Dịch Vụ)
  ↑ Số liệu đo lường thực tế
  ↑ Ví dụ: "availability = successful_requests / total_requests"
```

### SLI Phổ Biến Cho AWS Compute

| Loại SLI             | Công Thức                                                     | Ví Dụ Ngưỡng |
| -------------------- | ------------------------------------------------------------- | ------------ |
| **Availability**     | `successful_requests / total_requests × 100%`                 | SLO: 99.95%  |
| **Latency**          | `% requests hoàn thành trong X ms`                            | P99 < 500ms  |
| **Error Rate**       | `error_requests / total_requests × 100%`                      | < 0.1%       |
| **Throughput**       | `requests per second`                                         | > 1000 RPS   |
| **Saturation**       | `CPU/Memory utilization`                                      | < 70%        |

### Error Budget — Ngân Sách Lỗi

```
Error Budget = 1 - SLO

Ví dụ với SLO 99.9% (3 nines):
  Error Budget = 0.1% = 43.2 phút/tháng downtime được phép

Khi Error Budget cạn:
  → Freeze feature deployments
  → Tập trung vào reliability
  → Post-mortem bắt buộc
```

---

## Luồng Cảnh Báo

### Alert Pipeline Tiêu Chuẩn

```
┌─────────────────────────────────────────────────────────┐
│                   ALERT PIPELINE                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  METRIC ANOMALY                                         │
│       ↓                                                 │
│  CloudWatch Alarm (ALARM state)                         │
│       ↓                                                 │
│  SNS Topic (Simple Notification Service — Dịch Vụ Thông Báo) │
│       ↓ ─────────────────── ↓                          │
│  PagerDuty / OpsGenie       Lambda Auto-Remediation     │
│  (On-call Engineer)         (Tự Động Khắc Phục)        │
│       ↓                          ↓                      │
│  Investigate Runbook        Slack/Teams Notification    │
│       ↓                                                 │
│  Resolve & Post-mortem                                  │
└─────────────────────────────────────────────────────────┘
```

### Phân Cấp Alert — Alert Severity

| Mức Độ      | Ký Hiệu | Ví Dụ                             | Phản Ứng           |
| ----------- | ------- | --------------------------------- | ------------------ |
| **P1**      | 🔴      | Production down, data loss        | Wake on-call ngay  |
| **P2**      | 🟠      | Degraded performance, error spike | Alert trong 5 phút |
| **P3**      | 🟡      | Warning threshold crossed         | Next business hour |
| **P4**      | 🟢      | Informational, trend alert        | Weekly review      |

### Nguyên Tắc Chống Alert Fatigue (Mệt Mỏi Do Cảnh Báo)

1. **Actionable alerts only** — mỗi alert phải có người xử lý và biết làm gì
2. **Avoid flapping** — dùng evaluation periods đủ dài để tránh alert on/off liên tục
3. **Alert on symptoms, not causes** — alert "user can't checkout" tốt hơn "CPU > 80%"
4. **Regular review** — xóa alerts không có ai xử lý trong 30 ngày qua
5. **Composite alarms** — kết hợp điều kiện để giảm noise

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Thường Gặp

**Q: Sự khác biệt giữa CloudWatch Metrics và CloudWatch Logs?**

> Metrics là số liệu định lượng time-series (vd: CPUUtilization=78%), nhẹ và lưu 15 tháng. Logs là text records chi tiết từng sự kiện, phù hợp debug nhưng tốn storage hơn. Metric Filters có thể trích xuất metrics từ logs (vd: đếm số lần xuất hiện "ERROR" trong logs → tạo alarm).

**Q: Khi nào dùng X-Ray vs CloudWatch Logs để debug?**

> X-Ray dùng khi cần tìm **đâu** trong request flow gây chậm (latency breakdown qua nhiều services). CloudWatch Logs dùng khi cần biết **tại sao** một bước cụ thể thất bại (error messages, stack traces). Trong thực tế: X-Ray dẫn tới service có vấn đề → Log Insights tìm chi tiết trong service đó.

**Q: Composite Alarm là gì và khi nào dùng?**

> Composite Alarm kết hợp nhiều alarms với logic AND/OR. Dùng khi: muốn alert chỉ khi CPU > 80% VÀ Error rate > 5% (cùng lúc) — tránh false positive khi chỉ một chỉ số cao nhất thời.

**Q: Làm sao kiểm soát chi phí X-Ray?**

> Dùng sampling rules (quy tắc lấy mẫu): mặc định 5% requests được trace. Tăng sample rate cho paths quan trọng (checkout flow), giảm cho healthcheck endpoints. Dùng X-Ray Groups để filter và chỉ giữ traces có lỗi lâu hơn.

**Q: SLI, SLO, SLA khác nhau như thế nào?**

> SLI là số liệu đo thực tế (vd: latency P99 = 300ms). SLO là mục tiêu nội bộ (P99 < 500ms). SLA là cam kết với khách hàng trong hợp đồng (P99 < 1000ms). SLO luôn nghiêm ngặt hơn SLA để có buffer — nếu vi phạm SLO thì team biết trước vi phạm SLA.

---

## 🗺️ Điều Hướng

| File                          | Nội Dung                              |
| ----------------------------- | ------------------------------------- |
| [1-cloudwatch-metrics.md](./1-cloudwatch-metrics.md) | Metrics mặc định & tùy chỉnh       |
| [2-cloudwatch-logs.md](./2-cloudwatch-logs.md)       | Log Groups, Insights, Filters       |
| [3-cloudwatch-alarms.md](./3-cloudwatch-alarms.md)   | Alarms, Composite, Actions          |
| [4-xray-tracing.md](./4-xray-tracing.md)             | Distributed Tracing, Service Map    |
| [5-observability-checklist.md](./5-observability-checklist.md) | SLI/SLO/SLA, Runbooks      |

**← Trước:** [08-security/](../08-security/README.md) | **Tiếp theo →** [10-troubleshooting/](../10-troubleshooting/README.md)

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
