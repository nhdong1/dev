# CloudWatch Alarms & Composite Alarms

> **CloudWatch Alarms** (Cảnh Báo CloudWatch) giám sát metric theo thời gian thực và tự động kích hoạt hành động khi metric vượt ngưỡng định sẵn. **Composite Alarms** (Cảnh Báo Kết Hợp) kết hợp nhiều alarm bằng logic Boolean để giảm alarm fatigue (mệt mỏi cảnh báo).

---

## 📚 Mục Lục

1. [Alarm Là Gì?](#alarm-là-gì)
2. [Ba Trạng Thái Alarm](#ba-trạng-thái-alarm)
3. [Cấu Hình Alarm Chi Tiết](#cấu-hình-alarm-chi-tiết)
4. [Alarm Actions — Hành Động Tự Động](#alarm-actions--hành-động-tự-động)
5. [Composite Alarms — Cảnh Báo Kết Hợp](#composite-alarms--cảnh-báo-kết-hợp)
6. [Anomaly Detection Alarms](#anomaly-detection-alarms)
7. [Alarm Best Practices](#alarm-best-practices)
8. [Thiết Kế Hệ Thống Cảnh Báo](#thiết-kế-hệ-thống-cảnh-báo)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Alarm Là Gì?

**CloudWatch Alarm** theo dõi một **metric** (hoặc Metric Math expression) trong một **time period** (khoảng thời gian) và thực hiện **actions** (hành động) khi metric vượt ngưỡng (threshold) được định nghĩa.

```
Alarm = Metric + Threshold + Period + Actions

Ví dụ:
  Metric: EC2 CPUUtilization
  Threshold: > 80%
  Period: 5 phút
  Action: Gửi SNS notification → Slack/PagerDuty
```

### Alarm vs Notification

Alarm **không phải** là notification (thông báo) — alarm là trạng thái. Notification chỉ là một trong nhiều hành động mà alarm có thể kích hoạt (trigger). Alarm còn có thể trigger Auto Scaling, EC2 reboot, hay Lambda function.

---

## Ba Trạng Thái Alarm

```
             Không đủ dữ liệu
                    │
    ┌───────────────▼───────────────┐
    │       INSUFFICIENT_DATA       │
    │  (Không Đủ Dữ Liệu)          │
    └───────────────┬───────────────┘
                    │ Có đủ dữ liệu
          ┌─────────┴──────────┐
          │                    │
    ┌─────▼──────┐      ┌──────▼─────┐
    │     OK     │◄────►│   ALARM    │
    │  (Bình Thường)    │ (Cảnh Báo) │
    └────────────┘      └────────────┘
     metric ≤ threshold  metric > threshold
```

| Trạng Thái            | Ý Nghĩa                                              | Khi Nào Xảy Ra                                    |
| --------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| `OK`                  | Metric trong ngưỡng bình thường                     | Metric ≤ threshold (với GREATER_THAN_THRESHOLD)   |
| `ALARM`               | Metric vi phạm ngưỡng đủ số lần quy định           | Metric vượt threshold trong M trong N datapoints   |
| `INSUFFICIENT_DATA`   | Không đủ dữ liệu để đánh giá                        | Instance mới, metric chưa báo về, khoảng trống   |

### Lưu Ý Về INSUFFICIENT_DATA

`INSUFFICIENT_DATA` **không có nghĩa là lỗi**. Nó xảy ra khi:
- Instance mới khởi động, CloudWatch chưa nhận datapoint đầu tiên
- Metric không báo về trong một khoảng thời gian (ví dụ Lambda không được gọi)
- Alarm mới được tạo, chưa qua đủ evaluation period

Bạn có thể cấu hình alarm xử lý `INSUFFICIENT_DATA` như `OK` hoặc `ALARM` tùy context.

---

## Cấu Hình Alarm Chi Tiết

### Anatomy of an Alarm (Giải Phẫu Một Alarm)

```
┌─────────────────────────────────────────────────────────┐
│                    ALARM CONFIGURATION                  │
│                                                         │
│  Metric:        AWS/EC2 CPUUtilization                  │
│  Dimension:     InstanceId = i-0abc123                  │
│  Statistic:     Average                                 │
│  Period:        300 seconds (5 phút)                    │
│  Threshold:     > 80 Percent                            │
│  Comparison:    GreaterThanThreshold                    │
│  Evaluation:    3 of 5 datapoints in breach            │
│                 (3 trong 5 datapoint vi phạm → ALARM)  │
│  Missing Data:  breaching (hoặc notBreaching/ignore)   │
└─────────────────────────────────────────────────────────┘
```

### Các Tham Số Quan Trọng

**Period (Chu Kỳ):**
- Khoảng thời gian để tổng hợp datapoints thành một giá trị
- Thường: 60s, 300s (5 phút), 3600s (1 giờ)
- Với High-Resolution Metrics: có thể dùng 10s hoặc 30s

**Evaluation Period (Chu Kỳ Đánh Giá):**
- Tổng số period để đánh giá (N trong "M of N")
- Alarm nhìn vào N period gần nhất

**Datapoints to Alarm (Điểm Dữ Liệu Để Cảnh Báo):**
- M trong N — Số period vi phạm threshold để trigger ALARM
- **M of N pattern** tránh false positive từ spike ngắn

**Ví Dụ "M of N" Patterns:**

```
"1 of 1" → Ngay lập tức báo khi vi phạm (nhạy cảm nhất, nhiều false alarm)
"2 of 3" → Vi phạm 2 trong 3 period liên tiếp mới báo (cân bằng)
"3 of 5" → Vi phạm 3 trong 5 period (ít false alarm hơn, nhưng chậm hơn)
"5 of 5" → Phải vi phạm 5 period liên tiếp mới báo (ít nhạy nhất)

Thực tế:
  CPU spike 30 giây → "1 of 1" ALARM, "3 of 5" → OK (tốt!)
  CPU cao sustained 15 phút → Tất cả pattern đều ALARM
```

**Comparison Operators (Toán Tử So Sánh):**
- `GreaterThanThreshold` — metric > threshold
- `GreaterThanOrEqualToThreshold` — metric ≥ threshold
- `LessThanThreshold` — metric < threshold
- `LessThanOrEqualToThreshold` — metric ≤ threshold
- `GreaterThanUpperThreshold` / `LessThanLowerThreshold` — Dành riêng cho Anomaly Detection

**Missing Data Treatment (Xử Lý Dữ Liệu Thiếu):**

| Tùy Chọn        | Ý Nghĩa                                     | Dùng Khi                                |
| --------------- | ------------------------------------------- | --------------------------------------- |
| `missing`       | Không tính datapoint thiếu (mặc định)       | Metric báo không đều (Lambda, cron)     |
| `ignore`        | Bỏ qua, giữ nguyên trạng thái alarm        | Tương tự `missing`                      |
| `breaching`     | Coi như vi phạm threshold                   | Security alert — im lặng = nguy hiểm   |
| `notBreaching`  | Coi như không vi phạm                       | Báo chỉ khi có data và vi phạm         |

---

## Alarm Actions — Hành Động Tự Động

Khi alarm chuyển trạng thái, nó có thể trigger các actions:

### SNS Notification (Thông Báo SNS)

```
Alarm → SNS Topic → {
  Email subscription
  SMS subscription
  Lambda function (xử lý tùy chỉnh)
  SQS queue (queue cho downstream processing)
  HTTP/HTTPS endpoint (webhook)
  PagerDuty / OpsGenie / Slack (qua Lambda hoặc Chatbot)
}
```

### EC2 Actions

| Action              | Mô Tả                                    | Dùng Khi                        |
| ------------------- | ---------------------------------------- | ------------------------------- |
| Stop Instance       | Dừng instance                            | Tiết kiệm chi phí ngoài giờ làm |
| Terminate Instance  | Xóa instance                             | Cleanup tự động                 |
| Reboot Instance     | Khởi động lại                            | StatusCheckFailed               |
| Recover Instance    | Di chuyển sang hardware tốt hơn          | StatusCheckFailed_System        |

### Auto Scaling Actions

```
High CPU Alarm → Scale Out (thêm instance)
Low CPU Alarm  → Scale In (giảm instance)

Ví dụ policy:
  ALARM: CPUUtilization > 70% (3 of 5) → Add 2 instances
  ALARM: CPUUtilization < 30% (3 of 5) → Remove 1 instance
```

### Lambda Actions (Kể Từ 2023)

Alarm có thể trực tiếp invoke Lambda function mà không cần qua SNS — giảm latency và đơn giản hóa kiến trúc.

### Systems Manager OpsCenter

Alarm có thể tự động tạo **OpsItem** (Mục Vận Hành) trong SSM OpsCenter để track và resolve incidents (sự cố).

---

## Composite Alarms — Cảnh Báo Kết Hợp

### Vấn Đề Mà Composite Alarm Giải Quyết

Trong hệ thống production phức tạp, bạn có hàng trăm alarms. Không phải mọi alarm đơn đều đáng gây sự cố — một spike CPU ngắn không đáng gọi lúc 3 giờ sáng. Nhưng CPU cao + Memory thấp + Error rate tăng đồng thời thì phải báo ngay.

**Alarm fatigue** (Mệt Mỏi Cảnh Báo) xảy ra khi team nhận quá nhiều notification rác, bắt đầu bỏ qua chúng — và bỏ qua cả những alert thực sự quan trọng.

### Cách Composite Alarm Hoạt Động

```
Composite Alarm "AppUnhealthy" = 
  ALARM("HighCPU") AND ALARM("HighMemory") AND ALARM("HighErrorRate")
         ↑                   ↑                       ↑
    CPU > 80%          Memory > 90%           5xx errors > 5%

Kết quả:
  CPU spike 30s → HighCPU = ALARM, AppUnhealthy = OK (không báo)
  Memory leak dài hạn → HighMemory = ALARM, AppUnhealthy = OK (không báo)
  CPU + Memory cao + Errors → AppUnhealthy = ALARM → Báo PagerDuty
```

### Cú Pháp Composite Alarm Rule

```
# Cú pháp (Rule Expression)
ALARM("alarm-name-1") AND ALARM("alarm-name-2")
ALARM("alarm-name-1") OR  ALARM("alarm-name-3")
NOT ALARM("maintenance-mode")

# Ví dụ phức tạp:
ALARM("HighCPU") AND ALARM("HighMemory") AND NOT ALARM("MaintenanceWindow")
```

### Ví Dụ Thiết Kế Thực Tế

**Hệ Thống Thương Mại Điện Tử:**

```
┌─────────────────────────────────────────────────────────┐
│           COMPOSITE ALARM: "CriticalOutage"             │
│                                                         │
│  ALARM("OrderService-5xx-High")    ← > 5% error rate   │
│  AND                                                    │
│  ALARM("PaymentService-5xx-High")  ← > 1% error rate   │
│  AND NOT                                                │
│  ALARM("DeploymentInProgress")     ← suppress on deploy│
│                                                         │
│  → Action: PagerDuty P1, Wake up on-call engineer      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│           COMPOSITE ALARM: "PerformanceDegraded"        │
│                                                         │
│  ALARM("OrderService-P99-High")    ← P99 > 2s          │
│  OR                                                     │
│  ALARM("Database-ReplicaLag")      ← lag > 30s         │
│                                                         │
│  → Action: SNS → Slack #alerts-warning                 │
└─────────────────────────────────────────────────────────┘
```

### Giới Hạn Composite Alarms

- Tối đa **100 alarm** trong một Composite Alarm rule
- Có thể lồng Composite Alarm trong Composite Alarm (nested composite)
- Composite Alarm **không trigger** EC2 Actions hay Auto Scaling — chỉ SNS và Lambda

---

## Anomaly Detection Alarms

### CloudWatch Anomaly Detection (Phát Hiện Bất Thường)

Thay vì đặt threshold tĩnh (static), CloudWatch dùng **Machine Learning** (Học Máy) để học pattern bình thường của metric và alarm khi metric lệch khỏi expected band (dải kỳ vọng).

```
Normal pattern:
  Traffic cao: 09:00–18:00 (giờ làm việc)
  Traffic thấp: 18:00–09:00
  Traffic rất thấp: Cuối tuần

Static threshold: 1000 req/s → ALARM ban đêm dù bình thường
Anomaly Detection: Tự biết "đêm thứ Hai thường 200 req/s" → chỉ alert khi > 400 req/s vào đêm
```

### Cách Hoạt Động

1. CloudWatch phân tích **lịch sử metric 15 ngày** để học pattern
2. Tạo **expected band** (dải kỳ vọng) — upper và lower bound tự động điều chỉnh theo time-of-day, day-of-week
3. Alarm khi metric vượt ra ngoài band với **standard deviation** (độ lệch chuẩn) được cấu hình

```
# Ví dụ khai báo trong CloudFormation
AnomalyDetectorMetricStat:
  Metric:
    Namespace: AWS/ApplicationELB
    MetricName: RequestCount
  Stat: Average

Alarm:
  ComparisonOperator: GreaterThanUpperThreshold
  ThresholdMetricId: ad1  # Anomaly Detection band
  EvaluationPeriods: 3
  DatapointsToAlarm: 2
```

### Khi Nào Dùng Anomaly Detection vs Static Threshold

| Tình Huống                                  | Khuyến Nghị                   |
| ------------------------------------------- | ----------------------------- |
| Metric có pattern rõ ràng (business hours)  | Anomaly Detection             |
| Metric flat (không có seasonality)          | Static threshold đơn giản hơn |
| CPU > X% là luôn nguy hiểm                  | Static threshold              |
| Traffic spike so với baseline               | Anomaly Detection             |
| Error count (bất kỳ lỗi nào đều báo)       | Static threshold (threshold=0)|

---

## Alarm Best Practices

### 1. Tránh Alarm Fatigue

```
Nguyên Tắc:
  ✅ Alarm → Action có ý nghĩa (ai đó làm gì đó)
  ❌ Alarm → Nhận email, nhìn, bỏ qua (alarm rác)

Cách thực hiện:
  - Dùng M of N (ít nhất "2 of 3") thay vì "1 of 1"
  - Composite Alarms để chỉ alert khi thực sự critical
  - Phân cấp: P1 (PagerDuty ngay) vs P2 (Slack, sáng mai)
  - Review và loại bỏ alarms không có action định kỳ
```

### 2. Đặt Alarm Trên Symptoms, Không Phải Causes

```
❌ Alarm trên CPU > 80% (cause)
✅ Alarm trên Error Rate > 1% (symptom — user impact)
✅ Alarm trên P99 Latency > 2s (symptom — user impact)
✅ Alarm trên 5xx response count > 10/minute (symptom)

CPU cao có thể bình thường (batch job). Error rate cao thì luôn cần attention.
```

### 3. Chọn Đúng Statistic Cho Alarm

```
CPUUtilization     → Average (ngưỡng 70-80%)
NetworkErrors      → Sum (ngưỡng > 0 là có vấn đề)
Latency            → p99 (ngưỡng theo SLA)
FreeStorageSpace   → Minimum (tìm disk thấp nhất)
```

### 4. Cấu Hình Missing Data Phù Hợp

```
Security monitoring → "breaching" (im lặng = nguy hiểm)
Scheduled jobs      → "notBreaching" (không chạy không phải lỗi)
Lambda functions    → "missing" (không gọi = không có metric)
Continuous services → "breaching" (không báo về = có thể crash)
```

### 5. Naming Convention (Quy Ước Đặt Tên)

```
Gợi ý format: {Environment}-{Service}-{Metric}-{Severity}

Ví dụ:
  prod-orderservice-cpu-critical
  prod-orderservice-p99latency-warning
  prod-rds-replicalag-critical
  staging-paymentservice-5xx-warning
```

---

## Thiết Kế Hệ Thống Cảnh Báo

### Mô Hình Multi-Level Alerting (Cảnh Báo Đa Tầng)

```
┌──────────────────────────────────────────────────────────────┐
│                      ALERTING PYRAMID                        │
│                                                              │
│         ┌───────┐                                            │
│         │  P1   │ Critical — PagerDuty, wake on-call         │
│         │ Alarm │ Composite: multiple symptoms               │
│        ┌┴───────┴┐                                           │
│        │   P2    │ Warning — Slack #alerts-prod              │
│        │  Alarm  │ Single symptom, customer impact likely    │
│       ┌┴─────────┴┐                                          │
│       │    P3     │ Info — Dashboard + email digest          │
│       │   Alarm   │ Trending, proactive monitoring           │
│      ┌┴───────────┴┐                                         │
│      │   P4/None   │ Auto-heal — Scale, restart, remediate   │
│      │  Auto-action│ No human notification needed            │
│      └─────────────┘                                         │
└──────────────────────────────────────────────────────────────┘
```

### Kiến Trúc SNS Fan-out (Phân Phối SNS)

```
CloudWatch Alarm
       │
       ▼
   SNS Topic
  ┌────┴────────────────────────┐
  │                             │
  ▼                             ▼
Lambda Function             SQS Queue
(Slack/PagerDuty)           (Audit log,
                             ticket system)
       │
       ▼
  Slack #alerts
  PagerDuty P1
  Email to team
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa CloudWatch Alarm và EventBridge Rule?

| Tiêu Chí     | CloudWatch Alarm                      | EventBridge Rule                          |
| ------------ | ------------------------------------- | ----------------------------------------- |
| **Trigger**  | Metric threshold vi phạm              | Event pattern hoặc schedule               |
| **Input**    | Metric data (số)                      | Event (JSON object)                       |
| **Dùng khi** | "Alert khi CPU > X%"                  | "Khi instance terminated, làm Y"          |
| **Latency**  | ~1 phút (standard metrics)            | Gần real-time (< 1 giây)                 |

Kết hợp cả hai: Alarm kích hoạt SNS → SNS gửi vào EventBridge để fan-out phức tạp hơn.

### Q2: Composite Alarm có thể trigger Auto Scaling không?

**Không.** Composite Alarm chỉ support SNS và Lambda actions. Auto Scaling cần metric trực tiếp hoặc Simple/Step Scaling Policy từ alarm đơn.

Workaround: Composite Alarm → SNS → Lambda → cập nhật desired capacity thủ công qua API (nhưng phức tạp).

### Q3: Làm thế nào alarm ngay lập tức khi lần đầu vi phạm mà không chờ 5 phút?

Dùng **"1 of 1" evaluation** với period ngắn:
```
Period: 60s (hoặc 10s với High-Resolution Metrics)
Evaluation: 1 datapoint breaching → ALARM

Hoặc dùng High-Resolution Alarm với period 10s:
  → Phát hiện trong vòng 10 giây
```

### Q4: Alarm INSUFFICIENT_DATA có gửi notification không?

**Có**, nếu bạn cấu hình action cho state transition đến `INSUFFICIENT_DATA`. Mặc định alarm chỉ notify khi chuyển sang `ALARM`. Bạn có thể cấu hình notify riêng cho mỗi state transition.

### Q5: Tại sao nên dùng P99 thay vì Average cho Latency Alarm?

```
Scenario: 1000 requests/phút
  - 990 requests: 50ms (nhanh)
  - 10 requests: 5000ms (chậm do N+1 query)

Average = (990 * 50 + 10 * 5000) / 1000 = 99.5ms → Có vẻ OK
P99     = ~5000ms → Phát hiện vấn đề!

Đặt alarm trên Average 99.5ms → Không bao giờ alert
Đặt alarm trên P99 > 1000ms → ALARM ngay

1% user = trong hệ thống 1 triệu request/ngày = 10,000 user bị ảnh hưởng/ngày
```

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [1-metrics-namespaces.md](./1-metrics-namespaces.md) | [3-logs-insights.md](./3-logs-insights.md)
