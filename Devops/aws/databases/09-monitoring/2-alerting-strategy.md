# Alerting Strategy — Chiến Lược Cảnh Báo & SLO

> Hướng dẫn toàn diện về thiết lập Alerting Strategy (Chiến Lược Cảnh Báo) cho AWS Database Services — bao gồm SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ), thresholds (ngưỡng cảnh báo), alert routing (định tuyến cảnh báo), và cách tránh alert fatigue (mệt mỏi cảnh báo).

---

## 📚 Mục Lục

1. [SLO & SLA Fundamentals](#slo--sla-fundamentals)
2. [CloudWatch Alarms Cơ Bản](#cloudwatch-alarms-cơ-bản)
3. [Ngưỡng Chuẩn Cho Từng Dịch Vụ](#ngưỡng-chuẩn-cho-từng-dịch-vụ)
4. [Alert Routing & Escalation](#alert-routing--escalation)
5. [Chống Alert Fatigue](#chống-alert-fatigue)
6. [Composite Alarms](#composite-alarms)
7. [Runbook Integration](#runbook-integration)

---

## SLO & SLA Fundamentals

### Phân Biệt SLA, SLO, SLI

```
SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ):
  → Hợp đồng pháp lý với khách hàng
  → "Database uptime 99.9% hoặc hoàn tiền X%"
  → Hậu quả pháp lý / tài chính nếu vi phạm

SLO (Service Level Objective — Mục Tiêu Mức Dịch Vụ):
  → Mục tiêu nội bộ, thường nghiêm ngặt hơn SLA
  → "Chúng ta cần duy trì 99.95% để có buffer trước SLA 99.9%"
  → Team cam kết đạt được

SLI (Service Level Indicator — Chỉ Số Mức Dịch Vụ):
  → Metric cụ thể để đo SLO
  → "Tỷ lệ requests thành công / tổng requests"
  → Được đo từ CloudWatch metrics
```

### Ví Dụ SLO Cho Database

| SLO                          | SLI (Metric)                              | Mục Tiêu   | Đo Bằng          |
| ---------------------------- | ----------------------------------------- | ---------- | ---------------- |
| **Availability (Tính Sẵn Sàng)** | % thời gian database accessible         | 99.95%     | Synthetic checks |
| **Read Latency P99**         | 99th percentile read response time        | < 50ms     | ReadLatency p99  |
| **Write Latency P99**        | 99th percentile write response time       | < 100ms    | WriteLatency p99 |
| **Error Rate (Tỷ Lệ Lỗi)**   | % transactions failed                     | < 0.1%     | DatabaseErrors   |
| **Throttling Rate**          | % DynamoDB requests throttled             | < 0.01%    | ThrottledRequests|

### Error Budget — Ngân Sách Lỗi

```
Error Budget = 1 - SLO target

Ví dụ với SLO 99.95% availability:
  Error Budget / tháng (30 ngày):
    = (1 - 0.9995) × 30 × 24 × 60 = 21.6 phút/tháng

Khi Error Budget cạn kiệt:
  → Stop shipping new features
  → Focus 100% vào reliability
  → Post-mortem (phân tích sau sự cố) bắt buộc

→ Đây là nguyên tắc SRE (Site Reliability Engineering — Kỹ Thuật Độ Tin Cậy Trang Web) của Google
```

---

## CloudWatch Alarms Cơ Bản

### Anatomy of a CloudWatch Alarm (Cấu Trúc Cảnh Báo CloudWatch)

```
CloudWatch Alarm bao gồm:
  1. Metric        — Số liệu cần theo dõi
  2. Threshold     — Ngưỡng kích hoạt
  3. Evaluation    — Số datapoints cần vi phạm để alarm
  4. Period        — Khoảng thời gian đánh giá
  5. Action        — Hành động khi alarm (SNS, Auto Scaling...)

Ví dụ:
  Metric:     CPUUtilization
  Threshold:  > 85
  Evaluation: 3 of 5 datapoints (3 trong 5 điểm dữ liệu)
  Period:     60 seconds (1 phút)
  → Alarm kích hoạt khi CPU > 85% trong ít nhất 3 phút liên tiếp trong 5 phút
```

### Alarm States (Trạng Thái Cảnh Báo)

```
OK         — Metric trong ngưỡng cho phép, bình thường
ALARM      — Metric vượt ngưỡng, cần hành động
INSUFFICIENT_DATA — Không đủ dữ liệu để đánh giá (mới tạo / metric không có)

Transitions (Chuyển Đổi Trạng Thái):
  OK → ALARM: Kích hoạt SNS notification
  ALARM → OK: Kích hoạt SNS "RESOLVED" notification (nếu cấu hình)
```

### Tạo Alarm bằng AWS CLI

```bash
# Alarm cho RDS CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-prod-cpu-high" \
  --alarm-description "RDS CPU above 85% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=prod-mysql-01 \
  --period 60 \
  --evaluation-periods 5 \
  --datapoints-to-alarm 3 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --statistic Average \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-alerts \
  --ok-actions arn:aws:sns:ap-southeast-1:123456789:db-alerts

# Alarm cho DynamoDB Throttling
aws cloudwatch put-metric-alarm \
  --alarm-name "DynamoDB-orders-throttling" \
  --metric-name ThrottledRequests \
  --namespace AWS/DynamoDB \
  --dimensions Name=TableName,Value=orders \
  --period 60 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789:db-alerts
```

### Missing Data Treatment (Xử Lý Dữ Liệu Thiếu)

```
notBreaching  — Coi như OK (dùng cho metrics thỉnh thoảng không có dữ liệu)
breaching     — Coi như ALARM (cho high-security monitoring)
ignore        — Giữ trạng thái hiện tại
missing       — Chuyển sang INSUFFICIENT_DATA

Hướng dẫn chọn:
  FreeStorageSpace:  breaching (thiếu data = không biết = nguy hiểm)
  CPUUtilization:    notBreaching (instance có thể off, bình thường)
  ReplicaLag:        notBreaching (chỉ có khi có replica)
```

---

## Ngưỡng Chuẩn Cho Từng Dịch Vụ

### RDS / Aurora Alarm Thresholds

```
┌─────────────────────────────────┬───────────────┬───────────────┬─────────────────────────┐
│ Metric                          │ Warning       │ Critical      │ Ghi Chú                 │
├─────────────────────────────────┼───────────────┼───────────────┼─────────────────────────┤
│ CPUUtilization                  │ > 70% 10p     │ > 90% 5p      │ 5p = 5 phút liên tiếp   │
│ FreeableMemory                  │ < 25% RAM     │ < 10% RAM     │ Tính % từ instance spec  │
│ SwapUsage                       │ > 256 MB      │ > 1 GB        │ Bất kỳ swap = warning   │
│ FreeStorageSpace                │ < 20%         │ < 10 GB       │ Absolute và relative    │
│ DatabaseConnections             │ > 70% max     │ > 90% max     │ Xem max_connections     │
│ ReadLatency                     │ > 5ms         │ > 20ms        │ Dùng Average            │
│ WriteLatency                    │ > 5ms         │ > 20ms        │ Dùng Average            │
│ ReadIOPS                        │ > 80% prov    │ > 95% prov    │ % provisioned IOPS      │
│ WriteIOPS                       │ > 80% prov    │ > 95% prov    │                         │
│ ReplicaLag                      │ > 30s         │ > 5 phút      │ Chỉ Read Replicas       │
│ AuroraReplicaLag                │ > 1s          │ > 10s         │ Thường < 100ms          │
│ LoginFailures                   │ > 10/phút     │ > 50/phút     │ Dấu hiệu brute force    │
└─────────────────────────────────┴───────────────┴───────────────┴─────────────────────────┘
```

### DynamoDB Alarm Thresholds

```
┌─────────────────────────────────┬───────────────┬───────────────┬─────────────────────────┐
│ Metric                          │ Warning       │ Critical      │ Ghi Chú                 │
├─────────────────────────────────┼───────────────┼───────────────┼─────────────────────────┤
│ ConsumedRCU (% provisioned)     │ > 80%         │ > 95%         │ Chỉ Provisioned mode    │
│ ConsumedWCU (% provisioned)     │ > 80%         │ > 95%         │ Chỉ Provisioned mode    │
│ ThrottledRequests               │ > 0 (5p)      │ > 100/phút    │ Bất kỳ throttle = alert │
│ ReadThrottleEvents              │ > 0           │ > 50/phút     │                         │
│ WriteThrottleEvents             │ > 0           │ > 50/phút     │                         │
│ SuccessfulRequestLatency P99    │ > 50ms        │ > 100ms       │ Dùng p99 statistic      │
│ SystemErrors                    │ > 0           │ > 5           │ HTTP 500 từ DynamoDB    │
│ UserErrors                      │ > 10/phút     │ > 100/phút    │ HTTP 400 (client error) │
│ GlobalSecondaryIndexThrottles   │ > 0           │ > 20/phút     │ GSI cần capacity riêng  │
└─────────────────────────────────┴───────────────┴───────────────┴─────────────────────────┘
```

### ElastiCache Redis Alarm Thresholds

```
┌─────────────────────────────────┬───────────────┬───────────────┬─────────────────────────┐
│ Metric                          │ Warning       │ Critical      │ Ghi Chú                 │
├─────────────────────────────────┼───────────────┼───────────────┼─────────────────────────┤
│ CPUUtilization                  │ > 65%         │ > 90%         │ Redis = single-threaded  │
│ EngineCPUUtilization            │ > 50%         │ > 80%         │ Chỉ CPU cho Redis engine │
│ BytesUsedForCache (%)           │ > 75%         │ > 90%         │ Tính % maxmemory        │
│ Evictions                       │ > 0           │ > 100/phút    │ Cache thrashing signal  │
│ CurrConnections                 │ > 80% max     │ > 95% max     │                         │
│ ReplicationLag                  │ > 10s         │ > 60s         │ Replica nodes           │
│ SwapUsage                       │ > 50 MB       │ > 100 MB      │ Redis không nên dùng swap│
│ FreeableMemory                  │ < 25% total   │ < 10% total   │                         │
└─────────────────────────────────┴───────────────┴───────────────┴─────────────────────────┘
```

---

## Alert Routing & Escalation

### SNS Topic Architecture (Kiến Trúc Chủ Đề SNS)

SNS (Simple Notification Service — Dịch Vụ Thông Báo Đơn Giản) là trung tâm phân phối cảnh báo:

```
CloudWatch Alarms
      │
      ▼
  SNS Topics (Chủ Đề SNS)
  ├── db-critical     → PagerDuty on-call + Slack #incidents
  ├── db-warning      → Slack #db-alerts + Email
  ├── db-info         → Slack #db-monitoring (low priority)
  └── db-security     → Security team + Slack #security
```

### Severity Levels (Mức Độ Nghiêm Trọng)

```
P1 — Critical (Khẩn Cấp):
  Điều kiện: Database down, data loss risk, SLA breach imminent
  Action: Page on-call kỹ sư ngay lập tức (24/7)
  Response time: < 5 phút
  Ví dụ: FreeStorageSpace < 5 GB, DatabaseConnections > 95%

P2 — High (Cao):
  Điều kiện: Performance degraded significantly, approaching limits
  Action: Slack notification + email đến team
  Response time: < 30 phút
  Ví dụ: CPU > 85%, ThrottledRequests > 100/phút

P3 — Medium (Trung Bình):
  Điều kiện: Performance warnings, early indicators
  Action: Slack notification
  Response time: < 4 giờ (business hours)
  Ví dụ: CPU > 70%, ReplicaLag > 30s

P4 — Low (Thấp):
  Điều kiện: Informational, trends to watch
  Action: Email digest
  Response time: Xem xét trong tuần
  Ví dụ: Storage growing fast (nhưng còn nhiều), Cache hit rate giảm nhẹ
```

### Escalation Policy (Chính Sách Leo Thang)

```
Minute 0:   Alarm kích hoạt → Page on-call engineer
Minute 5:   Không acknowledge → Page backup on-call
Minute 15:  Không resolve → Page engineering manager
Minute 30:  Không resolve → Page VP Engineering + open war room

Công cụ escalation:
  - PagerDuty — Phổ biến nhất, tích hợp tốt với AWS
  - OpsGenie  — Alternative của Atlassian
  - VictorOps — Splunk product
```

### Notification Channels (Kênh Thông Báo)

```yaml
# Ví dụ cấu hình SNS subscriptions
sns_subscriptions:
  db-critical:
    - protocol: pagerduty
      endpoint: "https://events.pagerduty.com/..."
    - protocol: slack_webhook
      endpoint: "#incidents"
    - protocol: email
      endpoint: "oncall@company.com"

  db-warning:
    - protocol: slack_webhook
      endpoint: "#db-alerts"
    - protocol: email
      endpoint: "db-team@company.com"

  db-security:
    - protocol: slack_webhook
      endpoint: "#security-alerts"
    - protocol: email
      endpoint: "security@company.com"
```

---

## Chống Alert Fatigue

Alert fatigue (mệt mỏi cảnh báo) xảy ra khi quá nhiều cảnh báo không có ý nghĩa → kỹ sư bắt đầu ignore alerts → thực sự bỏ lỡ sự cố nghiêm trọng.

### Nguyên Tắc "Actionable Alerts" (Cảnh Báo Có Thể Hành Động)

```
Mỗi alert cần trả lời được 3 câu hỏi:
  1. Chuyện gì đang xảy ra? (What)
  2. Tại sao tôi nhận alert này? (Why)
  3. Tôi cần làm gì ngay bây giờ? (Action)

Nếu không trả lời được → Alert đó cần xem xét lại
```

### Evaluation Periods và Datapoints-to-Alarm

```
Sai lầm phổ biến:
  evaluation_periods: 1
  datapoints_to_alarm: 1
  → Alert ngay khi CPU spike 1 giây → Quá nhiều false positives

Đúng:
  evaluation_periods: 5
  datapoints_to_alarm: 3
  → Alert chỉ khi CPU cao liên tục 3 trong 5 phút

Hướng dẫn:
  - CPU/Memory: Cần 3-5 phút liên tiếp trước khi alert
  - Storage: Alert ngay (critical resource)
  - Throttling: Cần 2-3 phút (một chút throttling là bình thường)
  - Latency: Dùng percentile (p99) không phải average
```

### Alert Suppression và Maintenance Windows

```
Maintenance Window (Cửa Sổ Bảo Trì):
  - Lên lịch trước khi maintenance
  - Tạm dừng non-critical alarms
  - Chỉ giữ critical alarms active

Trong CloudWatch:
  aws cloudwatch disable-alarm-actions \
    --alarm-names "RDS-prod-cpu-high"

  # Sau maintenance
  aws cloudwatch enable-alarm-actions \
    --alarm-names "RDS-prod-cpu-high"

Tự động hóa với Lambda + EventBridge (Lịch Sự Kiện):
  - Tạo schedule: Tắt alarms lúc 02:00 - 04:00 AM Chủ Nhật
  - Bật lại sau maintenance window kết thúc
```

### Alert Deduplication (Loại Bỏ Trùng Lặp)

```
Vấn đề: 10 metrics cùng alert khi một instance down → 10 pages
Giải pháp: Composite Alarms (xem bên dưới) + PagerDuty alert grouping

PagerDuty Alert Grouping:
  - Gộp các alerts liên quan trong 5 phút vào 1 incident
  - Giảm noise khi có cascading failures
  - Context-based grouping (cùng DBInstanceIdentifier)
```

---

## Composite Alarms

Composite Alarms (Cảnh Báo Tổng Hợp) kết hợp nhiều alarms bằng logic AND/OR:

### Use Case 1: Chỉ Alert Khi Thực Sự Nghiêm Trọng

```
# Alert CPU cao NHƯNG ĐỒNG THỜI connections cũng đang tăng
CompositeAlarm: rds-needs-immediate-action
  ALARM("rds-cpu-critical") AND ALARM("rds-connections-high")

→ Tránh alert CPU cao khi đó chỉ là analytics batch job 3 AM
  (khi connections thấp, không có user traffic)
```

### Use Case 2: Database Unhealthy

```
# Database unhealthy nếu BẤT KỲ điều kiện nào
CompositeAlarm: rds-database-unhealthy
  ALARM("rds-cpu-critical")
  OR ALARM("rds-storage-critical")
  OR ALARM("rds-connections-full")
  OR ALARM("rds-replication-lag-critical")

→ 1 page duy nhất thay vì 4 pages riêng lẻ
```

### Tạo Composite Alarm bằng AWS CLI

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "rds-prod-critical" \
  --alarm-description "RDS needs immediate attention" \
  --alarm-rule "ALARM(\"rds-cpu-critical\") OR ALARM(\"rds-storage-critical\")" \
  --alarm-actions "arn:aws:sns:ap-southeast-1:123456789:db-critical" \
  --ok-actions "arn:aws:sns:ap-southeast-1:123456789:db-critical"
```

---

## Runbook Integration

### Runbook (Sổ Tay Xử Lý) là gì?

```
Runbook = Tài liệu step-by-step để xử lý một loại alert cụ thể

Mỗi alert nên có runbook link trong description:
  --alarm-description "RDS CPU > 90%. Runbook: https://wiki/db/cpu-high"
```

### Cấu Trúc Runbook Tốt

```markdown
# Runbook: RDS CPU > 90%

## Triệu Chứng
- CloudWatch alarm: RDS-prod-cpu-critical
- CPUUtilization > 90% trong 5 phút

## Impact (Tác Động)
- Query latency tăng cho tất cả users
- Nguy cơ connection timeout nếu kéo dài > 15 phút

## Immediate Actions (Hành Động Ngay)

### Bước 1: Xác Định Nguyên Nhân (5 phút)
1. Vào RDS Performance Insights
2. Xem Top SQL by CPU
3. Xác định query đang gây CPU cao

### Bước 2: Short-term Fix (Fix Ngắn Hạn)
Option A — Nếu là 1 query cụ thể:
  KILL query_id; -- MySQL
  SELECT pg_terminate_backend(pid); -- PostgreSQL

Option B — Nếu là workload tăng đột biến:
  - Chuyển read traffic sang Read Replica
  - Scale up instance (cần 15-20 phút)

### Bước 3: Verify Resolution (Xác Nhận Đã Xử Lý)
- CPU về dưới 60% trong 5 phút
- Latency bình thường

## Post-Incident
- Viết post-mortem nếu > 15 phút ảnh hưởng users
- Tạo Jira ticket để fix underlying cause
```

### AWS Systems Manager — Automation Runbooks

```
AWS cung cấp SSM Automation (Tự Động Hóa SSM) để chạy runbooks tự động:

Ví dụ: Tự động restart RDS parameter group khi memory > threshold
  → CloudWatch Alarm → SNS → Lambda → SSM Automation
  → Log kết quả vào S3
  → Notify team qua Slack
```

---

## Checklist Alerting Strategy

```
□ Định nghĩa SLO rõ ràng cho từng database service
□ Mỗi alarm có severity level (P1/P2/P3/P4)
□ Mỗi alert có runbook link trong description
□ Sử dụng evaluation_periods >= 3 để giảm false positives
□ Thiết lập escalation policy trong PagerDuty/OpsGenie
□ Tạo Composite Alarms cho related metrics
□ Thiết lập maintenance window schedule
□ Review alert noise hàng tuần (track false positive rate)
□ SLO breach alarm (cảnh báo khi gần hết error budget)
□ Test alarms hàng tháng (tạo alarm test, verify notification)
```

---

**Xem Trước:** [1-cloudwatch-dashboards.md](1-cloudwatch-dashboards.md) — Thiết Lập Dashboard
**Xem Tiếp:** [3-rds-enhanced-monitoring.md](3-rds-enhanced-monitoring.md) — Enhanced Monitoring

**Cập Nhật Lần Cuối:** 2026-05-15
