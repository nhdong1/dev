# 5 — Observability Checklist — Danh Mục Kiểm Tra Quan Sát Hệ Thống

> Observability (Khả Năng Quan Sát Hệ Thống) = Metrics + Logs + Traces + Dashboards + Runbooks + SLI/SLO/SLA — đây là checklist toàn diện để xây dựng hệ thống có thể tự giải thích trạng thái của nó

## 📚 Mục Lục

1. [SLI, SLO, SLA — Framework Đo Lường Chất Lượng](#sli-slo-sla-framework)
2. [Error Budget — Ngân Sách Lỗi](#error-budget)
3. [Dashboard Design — Thiết Kế Dashboard](#dashboard-design)
4. [Runbook — Sổ Tay Vận Hành](#runbook)
5. [On-call và Escalation Policy](#on-call-và-escalation)
6. [Observability Maturity Model](#observability-maturity-model)
7. [Checklist Trước Khi Production](#checklist-trước-khi-production)
8. [Post-Mortem Template](#post-mortem-template)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## SLI, SLO, SLA Framework

### Định Nghĩa Đầy Đủ

```
SLA — Service Level Agreement — Thỏa Thuận Mức Dịch Vụ
  → Cam kết pháp lý/hợp đồng với khách hàng
  → Vi phạm → bồi thường/hoàn tiền
  → Ví dụ: "AWS EC2 đảm bảo 99.99% uptime. Vi phạm → service credit"

SLO — Service Level Objective — Mục Tiêu Mức Dịch Vụ
  → Mục tiêu nội bộ team, NGHIÊM NGẶT HƠN SLA (có buffer)
  → Vi phạm → hành động cải thiện nội bộ
  → Ví dụ: "Order service ≥ 99.95% availability" (buffer 0.05% so với SLA)

SLI — Service Level Indicator — Chỉ Số Mức Dịch Vụ
  → Số liệu đo lường thực tế để tính SLO/SLA
  → Ví dụ: "availability = good_requests / total_requests"
```

### Quy Tắc Thiết Lập SLO

```
1. BẮT ĐẦU TỪ USER EXPERIENCE:
   "Người dùng cảm thấy thế nào?" → latency, availability, error rate

2. ĐO LẠT THỰC TẾ TRƯỚC:
   Chạy ứng dụng vài tuần → xem baseline thực tế → set SLO khả thi

3. NGƯỠNG PHẢI CÓ Ý NGHĨA:
   "99.9% availability" tốt hơn "99.99%" nếu team chưa mature enough
   SLO quá cao → luôn vi phạm → mất tác dụng

4. BUFFER GIỮA SLO VÀ SLA:
   SLA = 99.9% → SLO = 99.95% (buffer 0.05%)
   Đảm bảo có thời gian phản ứng trước khi vi phạm SLA

5. KHÔNG QUÁ NHIỀU SLO:
   Chọn 3-5 SLO quan trọng nhất → dễ theo dõi và cải thiện
```

### SLI Phổ Biến Cho AWS Compute

```
┌─────────────────────────────────────────────────────────────────────────┐
│  SLI TEMPLATE CHO TỪNG LOẠI SERVICE                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  API SERVICE (EC2, Lambda, ECS, EKS):                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Availability SLI = (Requests với status < 500) / Total Requests │   │
│  │ Latency SLI      = % Requests hoàn thành trong X ms            │   │
│  │ Error Rate SLI   = (5xx Errors) / Total Requests               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  DATA PIPELINE (Lambda + Kinesis, ECS + SQS):                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Freshness SLI   = Tuổi dữ liệu mới nhất được xử lý            │   │
│  │ Throughput SLI  = Records xử lý/giây                           │   │
│  │ Error Rate SLI  = Records xử lý thất bại / Total records       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  BATCH JOB (EC2, Lambda scheduled):                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Coverage SLI    = % jobs hoàn thành đúng hạn                  │   │
│  │ Correctness SLI = % output records đúng                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Ví Dụ SLO Hoàn Chỉnh — Order Service

```yaml
# SLO Document — Order Service
service: order-service
version: "1.0"
last_updated: "2026-05-15"

slos:
  - name: "Availability"
    description: "Tỷ lệ requests thành công"
    sli:
      metric: "(sum(rate(http_requests_total{status!~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))) * 100"
    objective: 99.9  # %
    window: 30d       # rolling 30 ngày
    alerting:
      warning: 99.95  # alert khi < 99.95%
      critical: 99.9  # SLO violation

  - name: "Latency P99"
    description: "99% requests hoàn thành trong 500ms"
    sli:
      metric: "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))"
    objective: 0.5    # 500ms
    window: 30d

  - name: "Error Rate"
    description: "Tỷ lệ lỗi dưới 0.1%"
    sli:
      metric: "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100"
    objective: 0.1    # %
    window: 30d
```

---

## Error Budget — Ngân Sách Lỗi

### Tính Error Budget

```
Error Budget = 1 - SLO Target

Ví dụ SLO 99.9% (3 nines):
  Error Budget = 100% - 99.9% = 0.1%

Đổi sang thời gian downtime cho phép/tháng:
  Tháng = 30 ngày × 24 giờ × 60 phút = 43,200 phút
  Error Budget = 0.1% × 43,200 = 43.2 phút/tháng

SLO        | Error Budget/tháng
99%        | 432 phút (7.2 giờ)
99.9%      | 43.2 phút
99.95%     | 21.6 phút
99.99%     | 4.32 phút
```

### Sử Dụng Error Budget Như Công Cụ Quyết Định

```
┌──────────────────────────────────────────────────────────────────┐
│              ERROR BUDGET DECISION FRAMEWORK                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Error Budget còn > 50%:                                        │
│  → Tự do deploy features mới                                    │
│  → Chấp nhận rủi ro hợp lý                                     │
│  → Focus vào velocity (tốc độ phát triển)                      │
│                                                                  │
│  Error Budget còn 0-50%:                                        │
│  → Cẩn thận hơn khi deploy                                     │
│  → Review deployment process                                    │
│  → Tăng cường testing                                          │
│                                                                  │
│  Error Budget đã cạn (0%):                                      │
│  → FREEZE tất cả feature deployments                           │
│  → Chỉ deploy reliability improvements                         │
│  → Bắt buộc post-mortem                                        │
│  → Investigate và fix underlying issues                        │
│                                                                  │
│  Error Budget âm (vi phạm SLO):                                 │
│  → Escalate lên leadership                                      │
│  → Xem xét SLO có realistic không                              │
│  → Dedicated reliability sprint                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Theo Dõi Error Budget Trong CloudWatch

```bash
# Tạo Metric Math expression cho Error Budget Burn Rate
# (Tốc Độ Tiêu Thụ Ngân Sách Lỗi)
# Burn Rate > 1 = đang tiêu budget nhanh hơn bình thường

# Ví dụ: Availability SLO 99.9%
# Error Budget = 0.1% = 0.001

# 1-hour burn rate:
# burn_rate_1h = (1 - availability_1h) / (1 - slo_target)
# Nếu burn_rate > 14.4 → budget cạn trong 5 ngày

# Alert khi burn rate cao bất thường
aws cloudwatch put-metric-alarm \
  --alarm-name "ErrorBudgetBurnRateCritical" \
  --alarm-description "Error budget burning 14x faster than normal — SLO at risk" \
  --metrics '[...]' \
  --threshold 14.4 \
  --comparison-operator GreaterThanThreshold
```

---

## Dashboard Design — Thiết Kế Dashboard

### Nguyên Tắc Thiết Kế Dashboard Hiệu Quả

```
1. PROGRESSIVE DISCLOSURE (Tiết Lộ Dần):
   Level 1 (Overview): Service health, SLO status, active incidents
   Level 2 (Drill-down): Per-component metrics, error breakdown
   Level 3 (Detail): Individual traces, log snippets

2. TOP-DOWN APPROACH:
   Business metrics → Service metrics → Infrastructure metrics
   (Không nhồi tất cả cùng một level)

3. ACTIONABLE WIDGETS ONLY:
   Mỗi widget phải có câu trả lời: "Nếu số này bất thường, tôi làm gì?"
   Widget không actionable → xóa đi

4. CONSISTENT TIME RANGES:
   Tất cả widgets trong cùng dashboard → cùng time range
   Sử dụng variable $__timeRange nếu dùng Grafana

5. COMPARE TO BASELINE:
   Thêm reference line cho SLO target
   Thêm comparison với tuần trước / tháng trước
```

### Dashboard Tiers — Phân Cấp Dashboard

```
TIER 1 — NOC Dashboard (Network Operations Center):
  → Hiển thị trên màn hình lớn trong phòng ops
  → Chỉ có green/yellow/red status
  → Số lớn, dễ nhìn từ xa
  → SLO compliance, active incidents, overall health

TIER 2 — Service Owner Dashboard:
  → Chi tiết hơn về 1 service cụ thể
  → Request rate, error rate, latency percentiles
  → Dependency health (databases, external APIs)
  → Error breakdown (top error types)

TIER 3 — Debug Dashboard:
  → Dùng khi đang investigate incident
  → Rất chi tiết, nhiều metrics
  → Pre-canned queries, log links, trace links
  → Internal metrics (queue depths, cache hit rates)
```

### CloudWatch Dashboard Mẫu — Order Service

```bash
# Tạo dashboard với JSON
aws cloudwatch put-dashboard \
  --dashboard-name "OrderService-Production" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "Request Rate & Error Rate",
          "metrics": [
            ["AWS/Lambda", "Invocations", "FunctionName", "order-handler", {"label": "Requests/min", "stat": "Sum", "period": 60}],
            ["AWS/Lambda", "Errors", "FunctionName", "order-handler", {"label": "Errors/min", "stat": "Sum", "period": 60, "color": "#d62728"}]
          ],
          "view": "timeSeries",
          "period": 60
        }
      },
      {
        "type": "metric",
        "properties": {
          "title": "Lambda Duration (P50/P95/P99)",
          "metrics": [
            ["AWS/Lambda", "Duration", "FunctionName", "order-handler", {"stat": "p50", "label": "P50", "color": "#2ca02c"}],
            ["...", {"stat": "p95", "label": "P95", "color": "#ff7f0e"}],
            ["...", {"stat": "p99", "label": "P99", "color": "#d62728"}]
          ]
        }
      },
      {
        "type": "alarm",
        "properties": {
          "title": "Active Alarms",
          "alarms": [
            "arn:aws:cloudwatch:ap-southeast-1:123456789012:alarm:OrderService-HighErrorRate",
            "arn:aws:cloudwatch:ap-southeast-1:123456789012:alarm:OrderService-HighLatency"
          ]
        }
      }
    ]
  }'
```

---

## Runbook — Sổ Tay Vận Hành

Runbook (Sổ Tay Vận Hành) là tài liệu step-by-step cho engineer on-call xử lý một loại sự cố cụ thể.

### Cấu Trúc Runbook Tiêu Chuẩn

```markdown
# Runbook: [Tên Alarm] — [Tên Service]

## Thông Tin Cơ Bản
- **Alarm**: `OrderService-HighErrorRate`
- **Severity**: P2 — Urgent
- **Owner**: Order Team
- **Last Updated**: 2026-05-15
- **Escalation**: #order-team-oncall → @order-team-lead (sau 15 phút)

## Mô Tả Sự Cố
Lambda `order-handler` có error rate > 1% trong 5 phút liên tiếp.

## Tác Động (Impact)
- Người dùng không đặt được đơn hàng
- Revenue impact: ~$X/phút downtime

## Điều Kiện Alarm
- Metric: `(Errors / Invocations) * 100 > 1`
- Period: 5 phút
- Evaluation: 2 of 3

## Quy Trình Xử Lý (Step-by-Step)

### Bước 1: Xác Nhận Sự Cố (2 phút)
1. Mở CloudWatch Dashboard: [link]
2. Kiểm tra Error Rate graph: đang tăng hay ổn định?
3. Kiểm tra Invocations: traffic bình thường hay spike?
4. Nếu error rate < 0.5% và giảm → có thể false alarm, monitor thêm 5 phút

### Bước 2: Chẩn Đoán (5 phút)
1. Mở CloudWatch Logs Insights: [link có pre-filled query]
   ```
   filter @type = "REPORT" OR @message like /ERROR/
   | sort @timestamp desc
   | limit 50
   ```
2. Xem error message phổ biến nhất
3. Mở X-Ray Service Map: [link] → tìm node đỏ
4. Click vào traces có lỗi → xem segment nào fail

### Bước 3: Xử Lý Theo Loại Lỗi

#### 3a. Connection Timeout đến Database
```bash
# Kiểm tra DynamoDB status
aws dynamodb describe-table --table-name Orders --query 'Table.TableStatus'

# Kiểm tra throttle
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ThrottledRequests \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```
→ Nếu throttled: tăng provisioned capacity hoặc switch sang On-Demand

#### 3b. Lambda Out of Memory (OOM)
```bash
# Xem memory usage trong logs
filter @message like /Runtime exited with error: signal: killed/
# → Tăng Lambda memory trong console hoặc CLI:
aws lambda update-function-configuration \
  --function-name order-handler \
  --memory-size 1024
```

#### 3c. External Payment API Lỗi
→ Kiểm tra status page payment provider: [link]
→ Nếu họ có incident: implement fallback (queue orders for retry)

### Bước 4: Escalate Nếu Không Resolve Được
- Sau 15 phút: ping @order-team-lead trên Slack #incidents
- Sau 30 phút: mở bridge call với team

## Recovery Verification
1. Kiểm tra Error Rate trở về < 0.1%
2. Kiểm tra Invocations bình thường
3. Test manual: gọi API đặt đơn hàng test
4. Monitor 15 phút sau khi ổn định

## Post-Incident Actions
- [ ] Post-mortem nếu downtime > 5 phút
- [ ] Update runbook nếu có bước mới
- [ ] Tạo ticket follow-up nếu cần fix dài hạn
```

---

## On-call và Escalation

### On-call Rotation Best Practices

```
1. ROTATION HỢP LÝ:
   - 1 tuần on-call mỗi N người (N = số người trong team)
   - Không on-call liên tiếp 2 tuần
   - Pair on-call: primary + secondary luôn sẵn sàng

2. HANDOFF PROCESS (Quy Trình Bàn Giao):
   - Cuộc họp handoff 30 phút cuối mỗi shift
   - Chia sẻ: incidents đang diễn ra, changes gần đây, context đặc biệt
   - Cập nhật runbooks nếu phát hiện thiếu sót

3. COMPENSATION (Bồi Thường):
   - Policy rõ ràng về compensatory time off sau on-call nặng
   - Alert sau giờ làm việc → tính thêm

4. BLAMELESS CULTURE (Văn Hóa Không Đổ Lỗi):
   - Post-mortem tập trung vào systems, không phải cá nhân
   - "Tại sao system cho phép điều này xảy ra?" thay vì "Ai gây ra?"
```

### Escalation Matrix

```
┌──────────────────────────────────────────────────────────────────┐
│                    ESCALATION MATRIX                             │
├──────────────┬────────────────────────────────────────────────── │
│ Thời Gian    │ Hành Động                                        │
├──────────────┼──────────────────────────────────────────────────┤
│ T+0          │ Alert fire → On-call engineer nhận               │
│ T+5 phút     │ Acknowledge trong PagerDuty (tránh escalate)     │
│ T+15 phút    │ Nếu chưa resolve → escalate lên Team Lead       │
│ T+30 phút    │ Nếu P1 → escalate lên Manager, mở bridge call   │
│ T+60 phút    │ Nếu P1 vẫn chưa resolve → escalate lên VP Eng  │
│ T+2 giờ      │ Thông báo cho khách hàng / customer success      │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## Observability Maturity Model

Đánh giá mức độ trưởng thành observability của hệ thống:

```
LEVEL 0 — BLIND (Mù):
  ✗ Không có monitoring
  ✗ Biết có vấn đề khi khách hàng báo cáo
  ✗ Debug = SSH vào server xem log thủ công

LEVEL 1 — REACTIVE (Phản Ứng):
  ✓ Basic CloudWatch metrics (CPU, memory)
  ✓ Email alert khi down
  ✗ Không có log aggregation
  ✗ Không có tracing

LEVEL 2 — PROACTIVE (Chủ Động):
  ✓ Centralized logging (CloudWatch Logs)
  ✓ Meaningful alerts với runbooks
  ✓ Basic dashboards
  ✓ Error tracking
  ✗ Không có distributed tracing

LEVEL 3 — OBSERVABLE (Có Thể Quan Sát):
  ✓ Tất cả Level 2
  ✓ Distributed tracing (X-Ray)
  ✓ Custom business metrics
  ✓ SLI/SLO defined và tracked
  ✓ Error budgets

LEVEL 4 — PROACTIVE + PREDICTIVE (Dự Báo):
  ✓ Tất cả Level 3
  ✓ Anomaly detection
  ✓ Capacity forecasting
  ✓ Automated remediation
  ✓ Chaos engineering
  ✓ AIOps insights
```

---

## Checklist Trước Khi Production

### Metrics Checklist

- [ ] EC2: CPU, Memory (CWAgent), Disk, StatusCheck alarms
- [ ] Lambda: Error rate, Duration P99, Throttles, DLQ alarms
- [ ] ECS: RunningTaskCount, CPU, Memory alarms
- [ ] Custom business metrics đã được publish
- [ ] Alarms có meaningful descriptions và runbook links
- [ ] Alarms test thủ công ít nhất 1 lần (dùng `set-alarm-state`)

### Logs Checklist

- [ ] Tất cả services đã gửi logs lên CloudWatch Logs
- [ ] Log Groups có retention policy (không để mặc định "Never Expire")
- [ ] Log format là JSON structured (không phải plain text)
- [ ] Correlation IDs có trong mỗi log entry
- [ ] Metric Filters đã tạo cho lỗi quan trọng
- [ ] Logs Insights queries đã saved cho scenarios phổ biến

### Tracing Checklist

- [ ] X-Ray đã bật trên tất cả services (Lambda, ECS, API Gateway)
- [ ] Trace ID propagation hoạt động qua tất cả services
- [ ] Custom annotations đã thêm cho business-critical context
- [ ] Sampling rules đã cấu hình (100% critical paths, thấp hơn cho healthchecks)
- [ ] Service Map đã verify là chính xác

### Dashboard Checklist

- [ ] Executive Dashboard: SLO status, incident count
- [ ] Service Dashboard: request rate, error rate, latency (P50/P95/P99)
- [ ] Infrastructure Dashboard: CPU, memory, network
- [ ] Link dashboards với nhau (drill-down path rõ ràng)

### SLO Checklist

- [ ] SLO document đã viết và đồng ý với stakeholders
- [ ] SLI metrics đang được thu thập chính xác
- [ ] Error budget dashboard đã tạo
- [ ] Alert khi burn rate quá cao
- [ ] Review SLO meeting schedule (hàng tháng)

### Runbooks Checklist

- [ ] Runbook cho mỗi P1/P2 alarm
- [ ] Runbook có step cụ thể, không chung chung
- [ ] Runbook có links đến dashboards, queries
- [ ] Runbook đã được test bởi engineer chưa biết hệ thống
- [ ] Runbook review định kỳ (sau mỗi incident)

---

## Post-Mortem Template

```markdown
# Post-Mortem: [Tên Sự Cố]

**Ngày:** 2026-05-15
**Duration:** 47 phút (14:23 – 15:10 UTC+7)
**Severity:** P2
**Services Ảnh Hưởng:** order-service, payment-service
**Author:** [Tên]
**Reviewers:** [Team Lead], [Senior Engineer]

## Timeline

| Thời Gian | Sự Kiện                                                |
| --------- | ------------------------------------------------------ |
| 14:23     | Alarm "OrderService-HighErrorRate" fire                |
| 14:25     | On-call engineer acknowledge                           |
| 14:30     | Xác định lỗi là timeout kết nối đến payment service   |
| 14:45     | Tìm root cause: payment service memory leak            |
| 15:05     | Restart payment service containers                     |
| 15:10     | Error rate về 0%, alarm OK                            |

## Root Cause (Nguyên Nhân Gốc Rễ)

Payment service có memory leak trong version 2.3.1 (deployed 11:00 ngày hôm đó). Sau 3.5 giờ, memory đạt 98%, GC pressure khiến response time > 30 giây, timeout từ order-service.

## Contributing Factors (Yếu Tố Góp Phần)

1. Deploy không có canary → 100% traffic ngay lập tức
2. Không có memory alarm cho payment service containers
3. Không có circuit breaker từ order-service đến payment-service
4. Load test không chạy đủ lâu để phát hiện memory leak

## Impact (Tác Động)

- 47 phút degraded service (orders chạy chậm/fail)
- ~2,300 failed order attempts
- Estimated revenue impact: $4,600

## What Went Well (Điều Tốt)

- Alarm phát hiện trong 3 phút (không đợi user báo cáo)
- Runbook giúp chẩn đoán nhanh
- Clear escalation path, team lead join trong 15 phút

## Action Items (Hành Động Tiếp Theo)

| Action                                          | Owner          | Due Date   |
| ----------------------------------------------- | -------------- | ---------- |
| Thêm memory alarm cho tất cả ECS services      | DevOps Team    | 2026-05-22 |
| Implement circuit breaker order→payment         | Order Team     | 2026-05-29 |
| Bắt buộc canary deployment cho payment service | Platform Team  | 2026-06-05 |
| Chạy load test ≥ 4 giờ để detect memory leaks | QA Team        | 2026-06-05 |
| Fix memory leak trong payment-service v2.3.2    | Payment Team   | 2026-05-20 |
```

---

## Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa SLO và SLA? Tại sao SLO phải nghiêm ngặt hơn SLA?**

> SLA (Thỏa Thuận Mức Dịch Vụ) là cam kết pháp lý với khách hàng — vi phạm → bồi thường. SLO (Mục Tiêu Mức Dịch Vụ) là mục tiêu nội bộ team. SLO phải nghiêm ngặt hơn SLA để có "buffer" — khi vi phạm SLO, team có thời gian phản ứng và sửa trước khi vi phạm SLA. Ví dụ: SLA = 99.9%, SLO = 99.95% → buffer 0.05% ≈ 21.6 phút/tháng.

**Q: Error Budget là gì và dùng như thế nào?**

> Error Budget = 1 - SLO Target. Với SLO 99.9% → Error Budget = 0.1% = 43.2 phút downtime/tháng được phép. Dùng như "ngân sách": khi budget còn nhiều → team thoải mái deploy features mới, chấp nhận rủi ro. Khi budget gần cạn → cẩn thận hơn, tăng cường testing. Khi cạn → freeze deployments, tập trung reliability. Đây là công cụ cân bằng giữa velocity và reliability.

**Q: Một good runbook cần có những gì?**

> Runbook tốt cần: (1) Mô tả rõ khi nào runbook này được dùng (alarm nào trigger). (2) Timeline xử lý rõ ràng với steps cụ thể, không chung chung "check the logs". (3) Links trực tiếp đến dashboards, pre-filled Logs Insights queries, Console pages. (4) Phân nhánh theo loại lỗi (case A làm X, case B làm Y). (5) Escalation path khi không tự xử lý được. (6) Verification steps để confirm đã resolve. Tiêu chuẩn kiểm tra: engineer chưa biết hệ thống có thể follow và resolve được không?

**Q: Làm sao thiết kế dashboard hiệu quả?**

> Nguyên tắc: (1) Top-down từ business → service → infrastructure. (2) Mỗi widget phải actionable — người xem biết làm gì khi số bất thường. (3) Progressive disclosure — overview → drill-down → detail. (4) SLO target làm reference line trên mỗi graph liên quan. (5) Hạn chế số widgets — 6-8 widgets per dashboard đủ. Sai lầm phổ biến: nhồi quá nhiều metrics, dashboard đẹp nhưng không dùng được lúc incident.

**Q: Bạn có kinh nghiệm cải thiện MTTD/MTTR không?**

> MTTD (Mean Time to Detect — Thời Gian Trung Bình Phát Hiện) giảm bằng: thêm meaningful alerts trước khi user báo cáo, anomaly detection. MTTR (Mean Time to Recover — Thời Gian Trung Bình Phục Hồi) giảm bằng: runbooks cụ thể, automation (auto-recovery, Lambda remediation), pre-validated rollback procedures, blameless post-mortems để học và cải thiện liên tục.

---

**← Trước:** [4-xray-tracing.md](./4-xray-tracing.md) | **Lên trên:** [README.md](./README.md)

**Cập Nhật Lần Cuối:** 2026-05-15 | **Phiên Bản:** 1.0
