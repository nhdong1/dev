# Cost Anomaly Detection — Phát Hiện Chi Phí Bất Thường

> **AWS Cost Anomaly Detection** (Phát Hiện Chi Phí Bất Thường) dùng **ML — Machine Learning** (Học Máy) để tự động học pattern chi tiêu lịch sử và phát hiện các khoản chi phí đột biến bất thường — không cần đặt ngưỡng thủ công như AWS Budgets.

---

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Anomaly Monitors — Bộ Theo Dõi](#anomaly-monitors--bộ-theo-dõi)
3. [Alert Subscriptions — Đăng Ký Cảnh Báo](#alert-subscriptions--đăng-ký-cảnh-báo)
4. [Cách ML Phát Hiện Anomaly](#cách-ml-phát-hiện-anomaly)
5. [So Sánh với AWS Budgets](#so-sánh-với-aws-budgets)
6. [Cấu Hình Thực Tế](#cấu-hình-thực-tế)
7. [Phân Tích Kết Quả Anomaly](#phân-tích-kết-quả-anomaly)
8. [Pricing](#pricing)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🔑 Khái Niệm Cốt Lõi

### Vấn Đề Cost Anomaly Detection Giải Quyết

```
Tình huống: Chi phí EC2 mọi thứ Hai thường $500.
  Thứ Hai này: $1,200.
  Nguyên nhân: Dev deploy script vô tình chạy loop tạo 50 instances.

AWS Budgets:
  Budget $5,000/tháng → 80% = $4,000
  Đến thứ Hai đó mới chi $1,500 tháng này
  → Budget alert CHƯA kích hoạt (chỉ 30%)
  → Anomaly không được phát hiện

Cost Anomaly Detection:
  Học rằng thứ Hai chi $500 là bình thường
  Phát hiện $1,200 là bất thường (+140%)
  → Alert ngay trong ngày
```

### Nguyên Lý Hoạt Động

```
BƯỚC 1: Học (Training)
  ↓
  AWS thu thập dữ liệu chi phí lịch sử
  ML model học pattern: daily/weekly seasonality,
  growth trends, known spikes

BƯỚC 2: Dự Đoán (Prediction)
  ↓
  Mỗi ngày: Model dự báo chi phí "expected" với confidence interval

BƯỚC 3: So Sánh (Detection)
  ↓
  Actual cost vs Expected cost
  Nếu actual vượt xa confidence interval → ANOMALY

BƯỚC 4: Cảnh Báo (Alert)
  ↓
  Gửi notification nếu anomaly vượt ngưỡng đã cấu hình
  (absolute $, % deviation, hoặc cả hai)
```

---

## 🔭 Anomaly Monitors — Bộ Theo Dõi

**Anomaly Monitor** (Bộ Theo Dõi Bất Thường) xác định **phạm vi** cần theo dõi. Mỗi monitor tập trung vào một góc nhìn cụ thể.

### 4 Loại Monitor

#### 1. AWS Service Monitor (Theo Dõi Theo Service)

```
Phạm vi: Toàn bộ account, phân tách theo từng AWS service
ML học: Pattern chi phí riêng của từng service

Ví dụ phát hiện:
  EC2: Chi phí tăng 200% trong 1 ngày
  S3: Data transfer đột biến (leak?)
  Lambda: Invocations tăng 10x (loop?)
  
Cấu hình: 1 monitor cover toàn bộ services
Use case: Phổ biến nhất — nên bật cho mọi account
```

#### 2. Linked Account Monitor (Theo Dõi Theo Account)

```
Phạm vi: Một hoặc nhiều member accounts trong Organization
ML học: Pattern chi phí riêng của từng account

Ví dụ phát hiện:
  Account dev-backend: Chi phí tăng 300% trong 2 ngày
  → Likely: dev quên tắt test environment lớn

Use case:
  → Management Account muốn monitor từng team account
  → Phát hiện account nào "đang cháy"
```

#### 3. Cost Category Monitor (Theo Dõi Theo Cost Category)

```
Phạm vi: Nhóm chi phí theo Cost Category
  (Cost Category = cách tổ chức chi phí theo rule tùy chỉnh)

Ví dụ:
  Cost Category "Production Workloads" gom:
    - EC2 in prod-account
    - RDS in prod-account
    - ELB in prod-account
  
  Monitor detect anomaly trong toàn bộ "Production Workloads"
  
Use case: Tổ chức theo business unit, không chỉ theo service/account
```

#### 4. Tag Monitor (Theo Dõi Theo Tag)

```
Phạm vi: Resources có tag key/value cụ thể

Ví dụ:
  Monitor: tag key = "team", tag value = "payment"
  → Theo dõi chi phí của mọi resource thuộc team payment
  → Alert khi payment team có chi phí bất thường

Use case:
  → Khi có tag strategy tốt
  → Monitor theo team/project thay vì account
```

### Tóm Tắt Chọn Monitor Type

```
Chỉ có 1 account                 → AWS Service Monitor
Multi-account, muốn per-account  → Linked Account Monitor
Đã có Cost Categories            → Cost Category Monitor
Tag strategy tốt, muốn per-team  → Tag Monitor
Tất cả tình huống lý tưởng       → Dùng nhiều monitor kết hợp
```

---

## 🔔 Alert Subscriptions — Đăng Ký Cảnh Báo

**Subscription** (Đăng Ký) xác định **ngưỡng** và **kênh** nhận cảnh báo.

### Threshold Configuration

```
Threshold options:
  1. Individual alert threshold (Ngưỡng từng cảnh báo):
     → Alert khi anomaly > $X hoặc > X%
     
  2. Combined threshold:
     → Alert khi anomaly > $X VÀ > X%
     → Giảm noise: tránh alert khi tăng $5 (10%) nhỏ không đáng kể
```

**Ví dụ cấu hình ngưỡng thực tế:**

```
Production environment:
  Amount: > $100  (absolute)
  Percentage: > 20%
  → Alert khi chi phí tăng > $100 VÀ > 20% so với expected

Dev environment:
  Amount: > $20
  Percentage: > 50%
  → Ngưỡng thấp hơn vì baseline nhỏ hơn
```

### Alert Frequency (Tần Suất Cảnh Báo)

| Frequency     | Khi Nào Nhận Alert                                          |
| ------------- | ----------------------------------------------------------- |
| **Individual** | Alert ngay khi phát hiện từng anomaly                      |
| **Daily**      | Tổng hợp tất cả anomaly trong ngày, gửi 1 lần             |
| **Weekly**     | Tổng hợp trong tuần — phù hợp alert ít quan trọng         |

### Notification Channels

```
1. Email — Đơn giản, không cần setup
   Gửi email với chi tiết: service nào, tăng bao nhiêu, thời điểm

2. SNS Topic — Linh hoạt
   → Lambda: Gửi Slack/Teams message với formatting đẹp
   → Lambda: Tạo JIRA ticket tự động
   → Lambda: Page on-call engineer qua PagerDuty
```

---

## 🤖 Cách ML Phát Hiện Anomaly

### Model Architecture

```
AWS dùng phương pháp Two-level model (Mô Hình Hai Tầng):

Tầng 1: Root-cause attribution (Quy Kết Nguyên Nhân)
  → Phân tách chi phí theo service/usage type/region
  → Mỗi dimension có model riêng

Tầng 2: Anomaly scoring (Chấm Điểm Bất Thường)
  → So sánh actual vs expected cho từng dimension
  → Tính anomaly score tổng hợp
```

### Các Yếu Tố ML Xem Xét

```
1. Day-of-week patterns (Pattern theo ngày trong tuần):
   Thứ Hai–Thứ Sáu: workload cao
   Cuối tuần: thấp hơn
   → Model biết "thứ Bảy chi $200 là bình thường"

2. Monthly patterns (Pattern theo tháng):
   Cuối tháng thường cao hơn đầu tháng
   → Model tính seasonality

3. Growth trends (Xu hướng tăng trưởng):
   Nếu cost tăng dần 5%/tháng → đây là expected growth, không phải anomaly

4. Known events:
   AWS biết service nào thường có spike theo kiểu nào
   (Ví dụ: S3 storage billing vào đầu tháng)
```

### Anomaly Score

```
Anomaly Score 0–100:
  0–49:   Low — Có thể là biến động bình thường
  50–74:  Medium — Cần xem xét
  75–100: High — Rất có thể là anomaly thực sự

Trong Console: Hiển thị cả impact ($) và score
→ Ưu tiên investigate high score + high $ impact trước
```

---

## ⚖️ So Sánh với AWS Budgets

| Tiêu Chí                    | AWS Budgets                          | Cost Anomaly Detection              |
| --------------------------- | ------------------------------------ | ----------------------------------- |
| **Cơ chế phát hiện**        | So sánh với ngưỡng cố định (static)  | ML học pattern động (dynamic)       |
| **Đặt ngưỡng thủ công**     | Bắt buộc ($X hoặc X%)               | Không cần (ML tự học)               |
| **Phát hiện spike ngắn hạn**| Kém — cần vượt % budget tổng        | Tốt — phát hiện bất thường trong ngày |
| **False positives**         | Ít (predictable)                    | Có thể cao trong giai đoạn đầu      |
| **Hành động tự động**       | Budget Actions (IAM, SCP, SSM)      | Chỉ notification (không có actions) |
| **Chi phí**                 | $0.02/budget/ngày                   | Miễn phí (alerts riêng có phí)      |
| **Use case**                | Kiểm soát ngân sách tháng           | Phát hiện sự cố bất ngờ             |

### Kết Hợp Tối Ưu

```
Cost Anomaly Detection:
  → Phát hiện spike bất thường trong ngày → điều tra ngay

AWS Budgets:
  → Kiểm soát tổng chi phí không vượt ngưỡng tháng
  → Trigger Budget Actions khi cần

Dùng cả hai = defense in depth (phòng thủ theo chiều sâu)
```

---

## 🔧 Cấu Hình Thực Tế

### Bước 1: Tạo Monitor

```
AWS Cost Management Console
  → Cost Anomaly Detection
  → Create monitor

Monitor name: "all-services-production"
Monitor type: AWS services
  → Tự động cover tất cả services trong account
```

### Bước 2: Tạo Alert Subscription

```
Alert name: "production-anomaly-alert"
Threshold:
  Individual alert threshold: $100 AND 20%
Frequency: Individual (real-time)

Recipients:
  Email: devops@company.com
  SNS: arn:aws:sns:us-east-1:123456789:cost-anomaly-topic
```

### Bước 3: Setup SNS → Slack (Tùy Chọn)

```python
# Lambda function: SNS → Slack notification

import json
import urllib.request

def lambda_handler(event, context):
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    anomaly = message['anomalyDetails']
    service = anomaly.get('rootCauses', [{}])[0].get('service', 'Unknown')
    impact = anomaly['impact']['totalImpact']
    start_date = anomaly['anomalyStartDate']
    
    slack_message = {
        "text": f"🚨 *Cost Anomaly Detected!*\n"
                f"*Service:* {service}\n"
                f"*Impact:* ${impact:.2f}\n"
                f"*Start:* {start_date}\n"
                f"*Action:* Check Cost Explorer for details"
    }
    
    webhook_url = "https://hooks.slack.com/..."
    data = json.dumps(slack_message).encode('utf-8')
    req = urllib.request.Request(webhook_url, data=data,
                                  headers={'Content-Type': 'application/json'})
    urllib.request.urlopen(req)
```

### CloudFormation Setup

```yaml
Resources:
  CostAnomalyMonitor:
    Type: AWS::CE::AnomalyMonitor
    Properties:
      MonitorName: all-services-monitor
      MonitorType: DIMENSIONAL
      MonitorDimension: SERVICE

  CostAnomalySubscription:
    Type: AWS::CE::AnomalySubscription
    Properties:
      SubscriptionName: production-alerts
      MonitorArnList:
        - !GetAtt CostAnomalyMonitor.MonitorArn
      Subscribers:
        - Address: devops@company.com
          Type: EMAIL
        - Address: !Sub "arn:aws:sns:${AWS::Region}:${AWS::AccountId}:cost-alerts"
          Type: SNS
      Threshold: 100
      ThresholdExpression:
        And:
          - Dimensions:
              Key: ANOMALY_TOTAL_IMPACT_ABSOLUTE
              Values: ["100"]
              MatchOptions: [GREATER_THAN_OR_EQUAL]
          - Dimensions:
              Key: ANOMALY_TOTAL_IMPACT_PERCENTAGE
              Values: ["20"]
              MatchOptions: [GREATER_THAN_OR_EQUAL]
      Frequency: IMMEDIATE
```

---

## 🔍 Phân Tích Kết Quả Anomaly

### Anatomy of an Anomaly Alert (Cấu Trúc Cảnh Báo)

```
Alert email chứa:
  ┌─────────────────────────────────────────────┐
  │ Cost Anomaly Detected                        │
  │                                             │
  │ Detection date: 2026-05-17                  │
  │ Anomaly start: 2026-05-16                   │
  │                                             │
  │ Total impact: $342.50                       │
  │ Expected spend: $85.20                      │
  │ Actual spend: $427.70                       │
  │                                             │
  │ Root cause:                                 │
  │   Service: Amazon EC2                       │
  │   Region: us-east-1                         │
  │   Usage type: BoxUsage:r5.4xlarge           │
  │                                             │
  │ [View in Cost Explorer]                     │
  └─────────────────────────────────────────────┘
```

### Quy Trình Điều Tra (Investigation Workflow)

```
1. Nhận alert → Xác định service/region bị ảnh hưởng

2. Mở Cost Explorer với link trong alert
   → Lọc theo service + region + ngày detect

3. Drill down vào Usage Type
   → Xác định resource nào gây ra (ví dụ: r5.4xlarge instances)

4. Xem CloudTrail để biết ai tạo resource đó
   → Event: RunInstances, khi nào, từ IP/user nào

5. Tắt resource không cần thiết

6. Root Cause Analysis:
   → Tại sao xảy ra?
   → Có thể ngăn chặn bằng gì? (SCP, Budget Action, Tag policy)

7. Mark anomaly là "resolved" hoặc "acknowledged" trong Console
```

---

## 💰 Pricing

```
Anomaly Monitors:     MIỄN PHÍ
Anomaly Detection:    MIỄN PHÍ

Alert Subscriptions:
  SNS:   $0.10/alert gửi qua SNS
  Email: MIỄN PHÍ

Thực tế:
  Số anomaly thường ít (vài lần/tháng)
  → Chi phí SNS rất thấp, thường < $1/tháng
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Cost Anomaly Detection khác AWS Budgets như thế nào?**
> Budgets dùng **static threshold** — alert khi vượt số tiền/% đã định trước. Cost Anomaly Detection dùng **ML dynamic baseline** — học pattern chi tiêu lịch sử và phát hiện khi thực tế khác xa baseline, kể cả khi chưa gần budget tổng. Anomaly Detection tốt hơn để phát hiện spike bất ngờ ngắn hạn; Budgets tốt hơn để enforce ngân sách cứng.

**Q: Có thể tự động stop resource khi phát hiện anomaly không?**
> Cost Anomaly Detection chỉ **gửi notification** (email hoặc SNS), không có built-in actions như Budget Actions. Để tự động hóa: SNS → Lambda → SSM Automation để stop resource. Hoặc kết hợp Budget Actions (set threshold thấp) để có automated remediation, dùng Anomaly Detection cho early warning.

**Q: Mất bao lâu để ML model hoạt động chính xác?**
> Model cần **ít nhất vài tuần đến 1 tháng** dữ liệu để học pattern. Trong giai đoạn đầu có thể có nhiều false positives. AWS khuyến nghị không chỉnh ngưỡng quá thấp trong 4–6 tuần đầu. Model tự cải thiện liên tục theo thời gian.

**Q: Tạo bao nhiêu monitor là hợp lý?**
> Thông thường: 1 **AWS Service Monitor** (cover tất cả services) + 1 **Linked Account Monitor** (nếu multi-account). Nếu có tag strategy tốt, thêm **Tag Monitor** theo team. Tránh tạo quá nhiều monitor với threshold thấp — dễ gây alert fatigue.

---

**Điều Hướng:**
← [2-cost-explorer.md](2-cost-explorer.md) | → [4-tagging-strategy.md](4-tagging-strategy.md)

**Cập Nhật Lần Cuối:** 2026-05-17
