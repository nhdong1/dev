# CloudWatch — Giám Sát & Quan Sát Toàn Diện

> **Amazon CloudWatch** là nền tảng quan sát (observability platform) trung tâm của AWS, thu thập metrics (chỉ số), logs (nhật ký), events (sự kiện) từ hơn 70 dịch vụ AWS và ứng dụng tùy chỉnh, cho phép vận hành hệ thống với tầm nhìn đầy đủ.

---

## 📚 Mục Lục

1. [Tại Sao Cần CloudWatch?](#tại-sao-cần-cloudwatch)
2. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
3. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
4. [Observability Strategy](#observability-strategy)
5. [Pricing Overview](#pricing-overview)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Điều Hướng Module](#điều-hướng-module)

---

## Tại Sao Cần CloudWatch?

Trong môi trường cloud, hạ tầng thay đổi liên tục — instance khởi động/tắt, container scale in/out, Lambda chạy vài millisecond rồi biến mất. Không có công cụ quan sát tập trung, bạn không thể:

- **Phát hiện sự cố sớm** — Biết CPU spike trước khi user báo lỗi
- **Phân tích nguyên nhân gốc rễ** (Root Cause Analysis — RCA) — Xem log ngay tại thời điểm sự cố
- **Tối ưu hiệu năng** — Phát hiện bottleneck (điểm nghẽn) qua dữ liệu
- **Tuân thủ SLA** (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) — Đo uptime, latency, error rate

CloudWatch trả lời ba câu hỏi cốt lõi của observability:

| Câu Hỏi            | Công Cụ CloudWatch             | Ví Dụ                                          |
| ------------------ | ------------------------------ | ---------------------------------------------- |
| **What happened?** | Metrics + Alarms               | CPU EC2 vượt 80% lúc 14:32                     |
| **Why happened?**  | Logs Insights                  | `ERROR OutOfMemoryError` trong log ứng dụng    |
| **How often?**     | Dashboards + Statistics        | P99 latency tăng mỗi thứ Hai giờ cao điểm     |

---

## Kiến Trúc Tổng Thể

```
┌─────────────────────────────────────────────────────────────┐
│                   DATA SOURCES (Nguồn Dữ Liệu)             │
│  EC2 │ RDS │ Lambda │ ECS/EKS │ ALB │ Custom App │ On-Prem │
└────────────────────────┬────────────────────────────────────┘
                         │  (CloudWatch Agent / SDK / API)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    CLOUDWATCH CORE                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   METRICS    │  │     LOGS     │  │     EVENTS       │  │
│  │  Namespaces  │  │  Log Groups  │  │  (EventBridge)   │  │
│  │  Dimensions  │  │  Log Streams │  │                  │  │
│  │  Statistics  │  │  Log Insights│  │                  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────┘  │
│         │                 │                                  │
│  ┌──────▼─────────────────▼──────────────────────────────┐  │
│  │                    ALARMS                             │  │
│  │  Simple Alarm │ Composite Alarm │ Anomaly Detection   │  │
│  └──────┬────────────────────────────────────────────────┘  │
│         │                                                    │
│  ┌──────▼──────────────────┐  ┌────────────────────────┐    │
│  │      DASHBOARDS         │  │   CONTAINER INSIGHTS   │    │
│  │  Widgets, Cross-account │  │   APPLICATION INSIGHTS │    │
│  └─────────────────────────┘  └────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    ACTIONS (Hành Động)                      │
│  SNS Notify │ Auto Scaling │ Lambda │ EC2 Action │ OpsCenter│
└─────────────────────────────────────────────────────────────┘
```

---

## Các Thành Phần Cốt Lõi

### 1. Metrics & Namespaces (Chỉ Số & Không Gian Tên)

**Metrics** là dữ liệu dạng số theo thời gian (time-series data). Mỗi metric được nhận dạng bởi:
- **Namespace** — Nhóm dịch vụ (`AWS/EC2`, `AWS/RDS`, `MyApp/Orders`)
- **Metric Name** — Tên chỉ số (`CPUUtilization`, `NetworkIn`, `OrderCount`)
- **Dimensions** (Chiều) — Bộ lọc bổ sung (`InstanceId=i-1234`, `Region=us-east-1`)
- **Resolution** (Độ Phân Giải) — Standard (60s) hoặc High-Resolution (1s)

**Retention (Lưu Trữ):**
- 1s – 3h: Lưu 3 giờ
- 60s: Lưu 15 ngày
- 5 phút: Lưu 63 ngày
- 1 giờ: Lưu 455 ngày (~15 tháng)

📄 **Chi tiết:** [1-metrics-namespaces.md](./1-metrics-namespaces.md)

---

### 2. Alarms & Composite Alarms (Cảnh Báo & Cảnh Báo Kết Hợp)

**Alarm** theo dõi một metric và thực hiện hành động khi vượt ngưỡng.

**Ba trạng thái:**
- `OK` — Metric trong ngưỡng bình thường
- `ALARM` — Metric vi phạm ngưỡng đã định
- `INSUFFICIENT_DATA` — Không đủ dữ liệu để đánh giá

**Composite Alarm** kết hợp nhiều alarm bằng logic `AND`/`OR`/`NOT`, giảm alarm noise (tiếng ồn cảnh báo) — chỉ báo động khi nhiều điều kiện cùng thỏa mãn.

📄 **Chi tiết:** [2-alarms-composite.md](./2-alarms-composite.md)

---

### 3. Logs & Logs Insights (Nhật Ký & Truy Vấn Nhật Ký)

**CloudWatch Logs** thu thập log từ:
- EC2 (qua CloudWatch Agent)
- Lambda (tự động)
- ECS/EKS Container Insights
- API Gateway, CloudTrail, VPC Flow Logs
- On-premises qua Agent

**Log Groups** (Nhóm Nhật Ký) — Container tổ chức log theo ứng dụng/môi trường.
**Log Streams** (Luồng Nhật Ký) — Chuỗi log từ một nguồn cụ thể (ví dụ: một EC2 instance).

**Logs Insights** — Ngôn ngữ truy vấn SQL-like để phân tích log nhanh.

📄 **Chi tiết:** [3-logs-insights.md](./3-logs-insights.md)

---

### 4. Dashboards & Widgets (Bảng Điều Khiển & Tiện Ích)

**CloudWatch Dashboards** — Bảng điều khiển tùy chỉnh, kết hợp nhiều metrics và logs từ nhiều region/account.

**Các loại Widget:**
- Line, Stacked Area, Number, Gauge, Bar
- Logs table (từ Logs Insights query)
- Alarm status, Text (Markdown)

**Cross-account & Cross-region Dashboard** — Quan sát toàn bộ multi-account từ một màn hình.

📄 **Chi tiết:** [4-dashboards-widgets.md](./4-dashboards-widgets.md)

---

### 5. Container Insights & Application Insights

**Container Insights** — Thu thập metrics và logs từ ECS, EKS, Kubernetes:
- CPU, memory, network, disk của từng pod/container
- Cluster-level và service-level aggregation

**Application Insights** — Tự động phát hiện sự cố và bottleneck trong ứng dụng .NET, Java, trên EC2 và ECS.

📄 **Chi tiết:** [5-container-application-insights.md](./5-container-application-insights.md)

---

### 6. CloudWatch Agent (Tác Nhân CloudWatch)

**CloudWatch Agent** là phần mềm cài trên EC2 (hoặc on-premises server) để:
- Thu thập **custom metrics** (chỉ số tùy chỉnh): RAM, disk, process count
- Thu thập **logs** từ file log tùy ý trên hệ thống
- Gửi dữ liệu lên CloudWatch với độ phân giải cao (High-Resolution: 1s)

📄 **Chi tiết:** [6-cloudwatch-agent.md](./6-cloudwatch-agent.md)

---

## Observability Strategy

### Ba Trụ Cột Của Observability (Quan Sát)

```
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│    METRICS     │  │     LOGS       │  │    TRACES      │
│  (Chỉ Số)     │  │  (Nhật Ký)    │  │  (Dấu Vết)    │
│                │  │                │  │                │
│ CloudWatch     │  │ CloudWatch     │  │  AWS X-Ray     │
│ Metrics        │  │ Logs           │  │  + OpenTel.    │
│                │  │                │  │                │
│ "Cái gì đang  │  │ "Tại sao xảy  │  │ "Request đi   │
│ xảy ra?"      │  │ ra?"           │  │ qua đâu?"     │
└────────────────┘  └────────────────┘  └────────────────┘
```

### Chiến Lược Monitoring Theo Tầng

| Tầng                    | Metrics Quan Trọng                              | Tool                        |
| ----------------------- | ----------------------------------------------- | --------------------------- |
| **Infrastructure**      | CPU, Memory, Disk, Network                      | CloudWatch Agent + Metrics  |
| **Application**         | Request rate, Error rate, Latency (P50/P95/P99) | Custom Metrics / EMF        |
| **Business**            | Orders/s, Revenue, Active users                 | Custom Metrics              |
| **Container**           | Pod CPU, Restart count, OOMKill                 | Container Insights          |
| **Database**            | Connections, Read/Write IOPS, Replication lag   | RDS/Aurora Metrics          |
| **Network**             | ALB 5xx rate, Target response time              | ALB Access Logs + Metrics   |

### USE Method & RED Method

**USE Method** (cho Infrastructure):
- **U**tilization — Mức sử dụng tài nguyên (CPU %)
- **S**aturation — Độ bão hòa (queue depth, throttling)
- **E**rrors — Số lỗi (error count, error rate)

**RED Method** (cho Services):
- **R**ate — Tốc độ request (requests/second)
- **E**rrors — Tỷ lệ lỗi (error %)
- **D**uration — Thời gian phản hồi (latency P99)

### Embedded Metric Format (EMF — Định Dạng Metric Nhúng)

EMF cho phép ghi metric trực tiếp trong log output mà không cần API gọi riêng:

```json
{
  "_aws": {
    "Timestamp": 1609459200000,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp",
      "Dimensions": [["Service"]],
      "Metrics": [{"Name": "OrderProcessingTime", "Unit": "Milliseconds"}]
    }]
  },
  "Service": "OrderService",
  "OrderProcessingTime": 245
}
```

CloudWatch tự động parse và đẩy metric từ log — không cần `PutMetricData` API call riêng.

---

## Pricing Overview

| Tính Năng                   | Free Tier                      | Giá (us-east-1)                    |
| --------------------------- | ------------------------------ | ---------------------------------- |
| **Metrics**                 | 10 custom metrics              | $0.30/metric/tháng (1–10k)         |
| **Alarms**                  | 10 alarms                      | $0.10/alarm/tháng                  |
| **Logs Ingestion**          | 5 GB/tháng                     | $0.50/GB                           |
| **Logs Storage**            | 5 GB/tháng                     | $0.03/GB/tháng                     |
| **Logs Insights Query**     | 5 GB scanned/tháng             | $0.005/GB scanned                  |
| **Dashboards**              | 3 dashboards (50 metrics each) | $3/dashboard/tháng                 |
| **High-Resolution Metrics** | -                              | $0.30/metric/tháng (same as basic) |

> **Lưu ý:** AWS Services như EC2, RDS tự động gửi metrics vào CloudWatch **miễn phí** — chỉ tính phí khi bạn tạo custom metrics hoặc dùng High-Resolution.

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: CloudWatch Metrics vs CloudWatch Logs — khác nhau như thế nào?

**Metrics** là dữ liệu số được tổng hợp theo thời gian (time-aggregated numeric data) — thích hợp để vẽ biểu đồ, đặt alarm, nhìn xu hướng. Metrics không chứa context chi tiết.

**Logs** là dữ liệu text thô ghi lại sự kiện cụ thể — thích hợp để debug, điều tra sự cố, phân tích nguyên nhân.

Kết hợp: Dùng Metric Alarm để phát hiện (CPU > 80%), dùng Logs Insights để tìm nguyên nhân (query log tìm exception).

---

### Q2: CloudWatch Alarm có bao nhiêu trạng thái và chuyển trạng thái như thế nào?

Ba trạng thái: `OK`, `ALARM`, `INSUFFICIENT_DATA`.

Alarm chuyển sang `ALARM` khi metric vi phạm ngưỡng đủ số lần trong evaluation period. Ví dụ: "CPU > 80% trong 3 trong 5 datapoints liên tiếp" — tránh false positive do spike ngắn.

`INSUFFICIENT_DATA` xảy ra khi metric chưa có đủ datapoints (instance mới khởi động, metric chưa báo về).

---

### Q3: Composite Alarm là gì và khi nào nên dùng?

**Composite Alarm** (Cảnh Báo Kết Hợp) kết hợp nhiều alarm đơn bằng `AND`/`OR`/`NOT`. Hữu ích để:

- **Giảm alarm fatigue** (mệt mỏi cảnh báo): Chỉ alert khi cả CPU cao VÀ memory thấp VÀ disk I/O cao đồng thời
- **Tạo health score tổng hợp**: Một alarm "application unhealthy" trigger khi bất kỳ critical component nào fail
- **Ưu tiên cảnh báo**: Phân loại severity (mức nghiêm trọng) tự động

---

### Q4: Làm thế nào để monitor RAM trên EC2?

EC2 **không tự động** gửi RAM metrics lên CloudWatch vì đây là thông tin bên trong OS, không phải hypervisor. Bạn cần:

1. Cài **CloudWatch Agent** trên instance
2. Cấu hình agent thu thập `mem_used_percent` từ OS
3. Agent gửi về namespace `CWAgent` (hoặc namespace tùy chỉnh)

---

### Q5: CloudWatch vs Prometheus/Grafana — khi nào dùng cái nào?

| Tiêu Chí              | CloudWatch                    | Prometheus + Grafana         |
| --------------------- | ----------------------------- | ---------------------------- |
| **Setup**             | Zero setup, tích hợp sẵn AWS  | Cần deploy và quản lý        |
| **Cost**              | Pay-per-use, có thể đắt       | Tự host, phần mềm miễn phí  |
| **AWS Integration**   | Native, không cần config      | Cần exporters                |
| **Flexibility**       | Giới hạn trong AWS ecosystem  | Rất linh hoạt, multi-cloud  |
| **Kubernetes**        | Container Insights            | Prometheus là chuẩn K8s      |
| **Best for**          | AWS-only stack, ít ops burden | Multi-cloud, K8s, flexibility|

---

## Điều Hướng Module

| File                                                                              | Nội Dung                                         |
| --------------------------------------------------------------------------------- | ------------------------------------------------ |
| [1-metrics-namespaces.md](./1-metrics-namespaces.md)                              | Metrics, Namespaces, Dimensions, Statistics      |
| [2-alarms-composite.md](./2-alarms-composite.md)                                  | Alarms, Composite Alarms, SNS actions            |
| [3-logs-insights.md](./3-logs-insights.md)                                        | Log Groups, Log Streams, Logs Insights queries   |
| [4-dashboards-widgets.md](./4-dashboards-widgets.md)                              | Dashboards, Widgets, Cross-account view          |
| [5-container-application-insights.md](./5-container-application-insights.md)      | Container Insights, Application Insights         |
| [6-cloudwatch-agent.md](./6-cloudwatch-agent.md)                                  | CloudWatch Agent, custom metrics, on-premises    |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
