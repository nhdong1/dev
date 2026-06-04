# AWS Budgets & Alerts — Ngân Sách và Cảnh Báo Chi Phí

> **AWS Budgets** cho phép đặt ngưỡng chi phí (cost threshold), lượng sử dụng (usage) và mức độ tận dụng cam kết (commitment utilization), sau đó kích hoạt cảnh báo (alert) hoặc hành động tự động (automated action) khi vượt ngưỡng.

---

## 📚 Mục Lục

1. [4 Loại Budget](#4-loại-budget)
2. [Cấu Hình Alert Thresholds](#cấu-hình-alert-thresholds)
3. [Budget Actions — Hành Động Tự Động](#budget-actions--hành-động-tự-động)
4. [Organization-Level Budgets](#organization-level-budgets)
5. [Giới Hạn và Pricing](#giới-hạn-và-pricing)
6. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 4 Loại Budget

### 1. Cost Budget (Ngân Sách Chi Phí)

**Mục đích:** Giới hạn tổng chi phí ($) trong kỳ (tháng, quý, năm).

```yaml
Budget Type: COST
Budget Amount: $500/month
Filters:
  - Service: Amazon EC2
  - Tag: Environment = prod
Alert Threshold: 80% actual, 100% forecasted
```

**Ví dụ thực tế:**
- Budget $1,000/tháng cho toàn account
- Alert khi đã chi $800 (80% actual)
- Alert khi dự báo sẽ vượt $1,000 (100% forecasted)

---

### 2. Usage Budget (Ngân Sách Sử Dụng)

**Mục đích:** Giới hạn lượng tài nguyên tiêu thụ, không phải chi phí.

```yaml
Budget Type: USAGE
Measure: GB-Month (S3 storage)
Budget Amount: 10,000 GB-Month
Filters:
  - Service: Amazon S3
  - Usage Type: TimedStorage-ByteHrs
```

**Các usage type phổ biến:**
| Service | Usage Measure | Đơn Vị |
|---------|---------------|--------|
| EC2 | Running hours | Hours |
| S3 | Storage | GB-Month |
| Data Transfer | Outbound | GB |
| Lambda | Invocations | Count |
| RDS | DB instance hours | Hours |

---

### 3. Reservation Budget — RI Utilization/Coverage

**Mục đích:** Đảm bảo Reserved Instances — RI (Phiên Bản Đặt Trước) được sử dụng hiệu quả.

```
Hai chỉ số theo dõi:

RI Utilization (Mức Độ Sử Dụng RI):
  = Số giờ RI thực sự chạy / Tổng số giờ RI đã mua
  Mục tiêu: > 80%
  Nếu thấp → đang lãng phí tiền mua RI

RI Coverage (Tỷ Lệ Bao Phủ RI):
  = Số giờ chạy được cover bởi RI / Tổng giờ chạy
  Mục tiêu: > 70%
  Nếu thấp → nhiều On-Demand đang chạy, nên mua thêm RI
```

**Alert khi utilization giảm dưới 80%:**
```yaml
Budget Type: RESERVATION
Coverage Type: UTILIZATION
Threshold: 80%
Alert Direction: LOWER_THAN  # Alert khi dưới ngưỡng
Services: EC2, RDS, ElastiCache
```

---

### 4. Savings Plans Budget

**Mục đích:** Tương tự RI Budget nhưng cho Savings Plans (Kế Hoạch Tiết Kiệm).

```
SP Utilization (Mức Độ Sử Dụng SP):
  = Commitment đã dùng / Tổng commitment đã cam kết
  Nếu thấp → đang commit quá nhiều, lãng phí cam kết

SP Coverage (Tỷ Lệ Bao Phủ SP):
  = Chi phí được cover bởi SP / Tổng chi phí compute
  Nếu thấp → nhiều On-Demand chưa được tiết kiệm
```

---

## Cấu Hình Alert Thresholds

### Loại Ngưỡng

```
Actual vs Forecasted:

  ACTUAL threshold:
    Kích hoạt khi chi phí thực tế ĐÃ vượt X%
    Ví dụ: Đã chi $800 = 80% của budget $1,000
    → Hành động: Thông báo ngay

  FORECASTED threshold:
    Kích hoạt khi AWS dự báo sẽ vượt X% vào cuối kỳ
    Ví dụ: Mới ngày 15 nhưng dự báo tháng này sẽ chi $1,100
    → Hành động: Cảnh báo sớm để điều chỉnh
```

### Cài Đặt Alert Điển Hình cho Production

```
Budget: $1,000/tháng

Alert 1: 50% Actual  → Email team lead
Alert 2: 80% Actual  → Email team lead + manager
Alert 3: 90% Actual  → Email + SNS → PagerDuty
Alert 4: 100% Forecasted → Email + Budget Action (restrict IAM)
Alert 5: 100% Actual → Email + SNS → Slack #cost-alerts
```

### Notification Channels

```yaml
Notifications:
  - Email: up to 10 addresses per alert
  - SNS Topic: trigger Lambda, Slack webhook, PagerDuty
  - AWS Chatbot: direct Slack/Teams integration
```

---

## Budget Actions — Hành Động Tự Động

**Budget Actions** (Hành Động Ngân Sách) cho phép AWS tự động thực hiện hành động khi vượt ngưỡng — không chỉ thông báo.

### 3 Loại Budget Action

#### Action 1: Apply IAM Policy (Áp Dụng Chính Sách IAM)

```
Khi budget vượt 100%:
  → Gắn IAM policy "DenyEC2Launch" vào role của team

IAM Policy "DenyEC2Launch":
{
  "Effect": "Deny",
  "Action": ["ec2:RunInstances", "ec2:StartInstances"],
  "Resource": "*"
}

Kết quả: Team không thể tạo thêm EC2 instance
         cho đến khi admin gỡ policy (hoặc budget reset)
```

#### Action 2: Apply Service Control Policy — SCP

```
Chỉ dùng được khi account nằm trong AWS Organizations

Khi budget Management Account vượt ngưỡng:
  → Gắn SCP vào OU hoặc account cụ thể
  → Chặn tất cả create/start resource trong account đó

Mạnh hơn IAM policy vì SCP không thể bị override
bởi bất kỳ permission nào trong account.
```

#### Action 3: Run AWS Systems Manager Action (Chạy SSM)

```
Khi budget vượt ngưỡng:
  → Kích hoạt SSM Automation document

Ví dụ SSM Document "StopNonProdEC2":
  1. List tất cả EC2 có tag Environment != prod
  2. Stop từng instance
  3. Gửi notification qua SNS

Use case: Budget vượt → tự động dừng dev instances
          giải phóng chi phí ngay lập tức
```

### Execution Type (Kiểu Thực Thi)

```
AUTOMATIC: AWS tự thực thi khi đạt threshold
           Không cần approval
           Dùng cho non-critical actions

REQUIRES_APPROVAL: AWS tạo action request
                   Admin phải approve trong Console/API
                   Dùng cho actions ảnh hưởng production
```

### Ví Dụ Kiến Trúc Budget Action

```
                    AWS Budgets
                        │
          Budget vượt 90% Actual
                        │
              ┌─────────▼──────────┐
              │   Budget Action    │
              │  (AUTO EXECUTE)    │
              └─────────┬──────────┘
                        │
           ┌────────────┼────────────┐
           │            │            │
    ┌──────▼──────┐ ┌───▼────┐ ┌────▼────────┐
    │  SNS Topic  │ │  IAM   │ │     SSM     │
    │  → Slack    │ │ Policy │ │  Automation │
    │  → Email    │ │ Attach │ │  Stop Dev   │
    └─────────────┘ └────────┘ │  Instances  │
                               └─────────────┘
```

---

## Organization-Level Budgets

### Budget Ở Mức Organization

```
Management Account có thể tạo budget cho:
  - Toàn bộ organization (consolidated)
  - Một member account cụ thể
  - Một Organizational Unit (OU)
  - Filtered by tag, service, region

Ví dụ:
  Budget "Dev-Accounts-Monthly" = $5,000
  Filter: OU = ou-xxxx-development
  → Kiểm soát tổng chi phí của tất cả dev accounts
```

### Delegated Budget Management

```
Cho phép member account tự quản lý budget:
  → Dùng Delegated Administrator pattern
  → Member account tự tạo budget cho mình
  → Management vẫn có consolidated view

Lưu ý: Mỗi account có thể tạo tối đa 20,000 budgets
```

---

## Giới Hạn và Pricing

### Free Tier

```
2 budgets đầu tiên: Miễn phí (mỗi tháng)
Budget thứ 3 trở đi: $0.02/budget/ngày (~$0.60/budget/tháng)

Ví dụ:
  10 budgets × $0.02 × 30 ngày = $4.80/tháng
  (Rất rẻ so với lợi ích kiểm soát chi phí)

Budget Actions: $0.10/action thực thi
```

### Giới Hạn Kỹ Thuật

| Giới Hạn | Giá Trị |
|----------|---------|
| Budgets tối đa mỗi account | 20,000 |
| Alert thresholds mỗi budget | 5 |
| Subscribers mỗi alert | 10 emails + 1 SNS |
| Budget Actions mỗi budget | 10 |
| Forecast lookback period | 5 tuần minimum |

---

## Thực Hành Tốt Nhất

### 1. Thiết Lập Budget Ngay Khi Có Account Mới

```
Checklist cho account mới:
  □ Budget tổng account $X/tháng (100% estimated spend)
  □ Alert 50%, 80%, 100% actual
  □ Alert 100% forecasted (cảnh báo sớm)
  □ SNS → Slack #aws-costs
  □ Budget Action tại 100%: notify manager
```

### 2. Budget Hierarchy (Phân Cấp Ngân Sách)

```
Level 1: Organization total    → $50,000/month
Level 2: OU (Production)       → $35,000/month
Level 3: OU (Development)      → $10,000/month
Level 4: Account (prod-us)     → $15,000/month
Level 5: Service (EC2 in prod) → $8,000/month
Level 6: Tag (team=payment)    → $3,000/month
```

### 3. Kết Hợp Forecasted và Actual

```
Chỉ dùng Actual alert:
  → Phát hiện sau khi đã tiêu quá nhiều
  → Phản ứng chậm

Kết hợp:
  Forecasted 80% → Early warning, điều chỉnh workload
  Actual 80%     → Chi phí đang cao, kiểm tra ngay
  Actual 100%    → Đã vượt, kích hoạt action
```

### 4. Tránh Alert Fatigue (Kiệt Sức Vì Cảnh Báo)

```
Alert fatigue: Quá nhiều alert → team bỏ qua → mất kiểm soát

Giải pháp:
  - Đặt ngưỡng hợp lý, không quá thấp
  - Route alert đúng người (dev alert → team lead, không phải mọi người)
  - Dùng actionable alert: "EC2 cost tăng 40%" thay vì chỉ "$800 spent"
  - Weekly summary thay vì alert từng ngày nhỏ
```

---

## Câu Hỏi Phỏng Vấn

### Q1: "Sự khác nhau giữa Cost Budget và Usage Budget?"

**Trả lời:**
> Cost Budget đo bằng tiền ($), giúp kiểm soát tổng chi phí. Usage Budget đo bằng đơn vị tài nguyên (GB, giờ, requests), hữu ích khi muốn kiểm soát mức tiêu thụ độc lập với giá — ví dụ khi giá spot thay đổi nhưng vẫn muốn giới hạn số giờ EC2 chạy.

### Q2: "Budget Actions có thể làm gì khi vượt ngưỡng?"

**Trả lời:**
> Ba loại action: (1) Gắn IAM policy để hạn chế quyền tạo tài nguyên; (2) Gắn SCP vào account/OU (trong Organizations) để chặn ở cấp organization; (3) Chạy SSM Automation document — ví dụ tự động stop non-prod EC2. Có thể chọn AUTOMATIC hoặc REQUIRES_APPROVAL tùy mức độ quan trọng.

### Q3: "Tại sao nên dùng cả Actual và Forecasted thresholds?"

**Trả lời:**
> Actual threshold phản ứng sau khi tiền đã tiêu — tốt để phát hiện vượt ngưỡng. Forecasted threshold phản ứng sớm dựa trên trend hiện tại — cho phép điều chỉnh workload trước khi vượt ngưỡng thực sự. Kết hợp cả hai cho phép both early warning và definitive action.

### Q4: "Giới hạn số budget mỗi account là bao nhiêu?"

**Trả lời:**
> 20,000 budget mỗi account — đủ cho mọi use case kể cả doanh nghiệp lớn. 2 budget đầu miễn phí, từ budget thứ 3 mất $0.02/ngày. Budget Actions có thêm phí $0.10/lần thực thi.

---

## 🔗 Liên Kết

- [2-cost-explorer.md](2-cost-explorer.md) — Phân tích chi phí sâu hơn
- [3-anomaly-detection.md](3-anomaly-detection.md) — Phát hiện tự động với ML
- [4-tagging-strategy.md](4-tagging-strategy.md) — Phân bổ chi phí theo tag

---

**Cập Nhật Lần Cuối:** 2026-05-17
