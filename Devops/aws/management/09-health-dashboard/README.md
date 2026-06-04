# AWS Health Dashboard — Sức Khỏe Dịch Vụ & Tự Động Hóa Phản Hồi

> **AWS Health Dashboard** là cổng thông tin tập trung giúp bạn theo dõi trạng thái sức khỏe của các dịch vụ AWS — từ sự cố toàn cầu đến các sự kiện ảnh hưởng trực tiếp đến tài nguyên trong account của bạn — và tự động hóa phản hồi thông qua EventBridge (Cầu Nối Sự Kiện).

---

## 📚 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Hai Chế Độ Chính](#hai-chế-độ-chính)
3. [Kiến Trúc & Luồng Dữ Liệu](#kiến-trúc--luồng-dữ-liệu)
4. [Các Loại Sự Kiện Health](#các-loại-sự-kiện-health)
5. [Tích Hợp EventBridge](#tích-hợp-eventbridge)
6. [Tích Hợp Multi-Account & Organizations](#tích-hợp-multi-account--organizations)
7. [So Sánh với CloudWatch & Trusted Advisor](#so-sánh-với-cloudwatch--trusted-advisor)
8. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
9. [Điều Hướng](#điều-hướng)

---

## Tổng Quan

**AWS Health Dashboard** (Bảng Điều Khiển Sức Khỏe AWS) được ra mắt như một sự thống nhất giữa hai công cụ trước đây:

| Tên cũ                              | Tên mới (2022+)                              | Mục đích                              |
| ----------------------------------- | -------------------------------------------- | ------------------------------------- |
| AWS Service Health Dashboard        | **Service Health** tab                       | Trạng thái công khai toàn bộ dịch vụ |
| AWS Personal Health Dashboard (PHD) | **Your Account Health** tab                  | Sự kiện ảnh hưởng account của bạn    |

> **Ghi nhớ phỏng vấn:** "Service Health" = public/global; "Account Health" = private/account-specific.

---

## Hai Chế Độ Chính

### 1. Service Health (Sức Khỏe Dịch Vụ Toàn Cầu)

Đây là trang công khai mà **bất kỳ ai** đều có thể truy cập tại `health.aws.amazon.com`.

**Đặc điểm:**
- Hiển thị trạng thái của **toàn bộ dịch vụ AWS** trên mọi region
- Cập nhật theo thời gian thực khi có sự cố (incident), bảo trì (maintenance), hoặc phục hồi (recovery)
- Lịch sử sự kiện có thể xem lại
- Không yêu cầu đăng nhập AWS

**Hạn chế:**
- Chỉ hiển thị sự cố **quy mô lớn** ảnh hưởng nhiều khách hàng
- Không cho biết **tài nguyên cụ thể nào** của bạn bị ảnh hưởng

```
Ví dụ: "EC2 tại us-east-1 đang gặp sự cố kết nối"
→ Bạn biết AWS đang có vấn đề nhưng không biết instance nào của mình bị ảnh hưởng
```

---

### 2. Account Health / Personal Health Dashboard — PHD (Bảng Điều Khiển Sức Khỏe Cá Nhân)

Truy cập qua AWS Console → Health Dashboard → **Your Account Health**.

**Đặc điểm:**
- Hiển thị sự kiện ảnh hưởng **trực tiếp đến tài nguyên trong account của bạn**
- Thông báo **chủ động** (proactive notification) trước khi bảo trì xảy ra
- Cung cấp **hướng dẫn hành động** (recommended actions) cụ thể
- Lưu trữ lịch sử sự kiện 90 ngày

**Các loại thông báo điển hình:**
- EC2 instance sắp được retire (thu hồi phần cứng)
- RDS certificate sắp hết hạn
- EBS volume bị suy giảm hiệu suất
- Lambda runtime sắp kết thúc hỗ trợ (end-of-life)
- Bảo trì có lịch ảnh hưởng đến VPC của bạn

---

## Kiến Trúc & Luồng Dữ Liệu

```
AWS Health Service
       │
       ├─── Service Health (Public)
       │         └── health.aws.amazon.com
       │
       └─── Account Health (Private)
                 │
                 ├── Console: AWS Health Dashboard → Your Account Health
                 │
                 ├── API: AWS Health API (chỉ Business/Enterprise Support)
                 │
                 └── EventBridge ──► Lambda / SNS / SQS / Step Functions
                                           │
                                           ├── Slack notification
                                           ├── PagerDuty / OpsGenie
                                           ├── Auto-remediation
                                           └── Ticket creation (JIRA, ServiceNow)
```

> **Lưu ý quan trọng:** AWS Health API và EventBridge integration **yêu cầu AWS Business Support hoặc Enterprise Support**. Tài khoản Developer/Basic không có quyền truy cập programmatic.

---

## Các Loại Sự Kiện Health

### Phân Loại Theo Phạm Vi

| Loại                                         | Mô Tả                                                           | Ví Dụ                                                  |
| -------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------ |
| **Account-specific events** (Sự kiện cụ thể) | Ảnh hưởng đến tài nguyên xác định trong account của bạn       | "EC2 instance i-abc123 trên phần cứng sắp retire"     |
| **Public events** (Sự kiện công khai)        | Sự cố AWS ảnh hưởng nhiều khách hàng, được phản ánh trên PHD  | "Degraded EC2 API calls tại ap-southeast-1"            |

### Phân Loại Theo Loại Sự Kiện

| `eventTypeCategory`                   | Mô Tả Tiếng Việt                          | Đặc Điểm                                     |
| ------------------------------------- | ----------------------------------------- | -------------------------------------------- |
| **issue** (Sự Cố)                     | Sự cố đang xảy ra ảnh hưởng dịch vụ      | Cần phản hồi ngay, có thể kéo dài            |
| **scheduledChange** (Thay Đổi Lịch)  | Bảo trì có lịch hoặc hardware retirement | Thông báo trước, có thời gian chuẩn bị       |
| **accountNotification** (Thông Báo)  | Thông báo liên quan đến account/billing  | Thông tin, không ảnh hưởng trực tiếp service |

### Vòng Đời Sự Kiện (Event Status)

```
open → upcoming → closed
  │                  │
  └─ (issue đang     └─ (đã giải quyết hoặc
      diễn ra)           bảo trì hoàn thành)
```

---

## Tích Hợp EventBridge

**EventBridge** (Cầu Nối Sự Kiện) là cách chính để tự động hóa phản hồi với Health events.

### Cấu Trúc Event Bridge Rule cho Health

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"],
  "detail": {
    "service": ["EC2", "RDS", "LAMBDA"],
    "eventTypeCategory": ["issue", "scheduledChange"],
    "eventTypeCode": ["AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED"]
  }
}
```

### Các Pattern Phổ Biến

#### Pattern 1: Cảnh Báo Slack Khi Có Sự Cố

```
Health Event → EventBridge Rule → SNS Topic → Lambda → Slack Webhook
```

**Chi tiết Lambda handler:**
```python
import json, urllib.request

def handler(event, context):
    detail = event['detail']
    service = detail['service']
    event_type = detail['eventTypeCode']
    affected = detail.get('affectedEntities', [])

    message = f"⚠️ AWS Health Alert\nService: {service}\nEvent: {event_type}\nAffected: {len(affected)} resources"

    # Gửi tới Slack webhook
    payload = json.dumps({"text": message}).encode()
    req = urllib.request.Request(SLACK_WEBHOOK_URL, data=payload)
    urllib.request.urlopen(req)
```

#### Pattern 2: Tự Động Migrate EC2 Trước Retirement

```
Health Event (AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED)
    → EventBridge Rule
    → Step Functions (Máy Trạng Thái)
        ├── Bước 1: Snapshot EBS volume
        ├── Bước 2: Stop instance
        ├── Bước 3: Launch replacement instance
        └── Bước 4: Update DNS / Load Balancer
```

#### Pattern 3: Tạo Ticket Tự Động

```
Health Event → EventBridge Rule → Lambda → JIRA/ServiceNow API
                                         → Tag EC2 instance với incident ID
                                         → Notify on-call engineer qua PagerDuty
```

### Thiết Lập EventBridge Rule (Terraform Example)

```hcl
resource "aws_cloudwatch_event_rule" "health_events" {
  name        = "capture-aws-health-events"
  description = "Capture AWS Health events for automation"

  event_pattern = jsonencode({
    source      = ["aws.health"]
    detail-type = ["AWS Health Event"]
    detail = {
      eventTypeCategory = ["issue", "scheduledChange"]
    }
  })
}

resource "aws_cloudwatch_event_target" "notify_lambda" {
  rule      = aws_cloudwatch_event_rule.health_events.name
  target_id = "SendToLambda"
  arn       = aws_lambda_function.health_processor.arn
}
```

---

## Tích Hợp Multi-Account & Organizations

### AWS Health với Organizations

Khi bật **AWS Organizations integration**, bạn có thể xem Health events của **toàn bộ member accounts** từ management account.

```
Management Account
    └── Health Dashboard (Organizational View)
            ├── Account A: 2 active events
            ├── Account B: 0 events
            └── Account C: 1 scheduled maintenance
```

**Yêu cầu kích hoạt:**
1. Tài khoản phải có **Business hoặc Enterprise Support** (ít nhất management account)
2. Bật tích hợp: `aws health enable-health-service-access-for-organization`
3. Region: Health API chỉ hoạt động tại **us-east-1** (global endpoint)

### Delegated Administrator cho Health

Có thể ủy quyền (delegate) cho một member account để quản lý Health events thay vì dùng management account trực tiếp.

```bash
# Từ management account
aws organizations register-delegated-administrator \
  --account-id 123456789012 \
  --service-principal health.amazonaws.com
```

### EventBridge Cross-Account cho Health Events

Kiến trúc tập trung (centralized) cho multi-account:

```
Account A ──┐
Account B ──┤──► EventBridge Bus (tập trung tại Security Account)
Account C ──┘         └─► Lambda / SQS / SNS (xử lý tập trung)
                              └─► SIEM / PagerDuty / Ops dashboard
```

---

## So Sánh với CloudWatch & Trusted Advisor

| Tiêu Chí                  | AWS Health Dashboard              | CloudWatch                          | Trusted Advisor                      |
| ------------------------- | --------------------------------- | ----------------------------------- | ------------------------------------ |
| **Nguồn dữ liệu**         | AWS platform events               | Metrics/Logs từ tài nguyên của bạn | Phân tích cấu hình tài nguyên        |
| **Câu hỏi trả lời**       | "AWS có vấn đề gì ảnh hưởng tôi?" | "Ứng dụng tôi hoạt động thế nào?"  | "Tôi đang làm gì sai so với best practice?" |
| **Tần suất cập nhật**     | Real-time (sự kiện AWS)           | Real-time (poll theo period)        | Định kỳ (refresh hàng tuần/theo yêu cầu) |
| **Yêu cầu Support Plan**  | Business+ cho API/EventBridge     | Không (miễn phí cơ bản)            | Business+ cho full checks            |
| **Tự động hóa**           | EventBridge integration           | Alarms → SNS/Lambda                 | EventBridge integration              |
| **Phạm vi**               | Infrastructure AWS platform       | Application & resource metrics      | Cost, Security, Performance, FT      |

> **Câu chốt phỏng vấn:**
> - CloudWatch = bạn giám sát tài nguyên của mình
> - Health Dashboard = AWS thông báo cho bạn về vấn đề của họ ảnh hưởng đến bạn
> - Trusted Advisor = AWS khuyên bạn cải thiện cấu hình

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Khác biệt giữa Service Health và Personal Health Dashboard là gì?

**Trả lời:**
- **Service Health** (public): Ai cũng xem được, hiển thị sự cố AWS quy mô lớn, không biết tài nguyên nào của bạn bị ảnh hưởng
- **Personal Health Dashboard / Account Health** (private): Chỉ bạn thấy, hiển thị sự kiện cụ thể ảnh hưởng đến tài nguyên trong account của bạn, kèm hướng dẫn hành động

---

### Q2: Làm sao tự động phản hồi khi EC2 instance sắp bị retire?

**Trả lời:** Dùng EventBridge bắt sự kiện `AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED` từ nguồn `aws.health`, trigger Lambda hoặc Step Functions để:
1. Chụp snapshot EBS
2. Dừng instance cũ
3. Khởi động instance thay thế trên phần cứng mới
4. Cập nhật load balancer / DNS

---

### Q3: Health API có yêu cầu gì đặc biệt không?

**Trả lời:** Có hai yêu cầu:
1. **AWS Business Support hoặc Enterprise Support** — tài khoản Basic/Developer không gọi được API
2. **Region us-east-1** — Health API là global endpoint, chỉ hoạt động ở us-east-1 dù tài nguyên ở region khác

---

### Q4: Làm sao xem Health events của nhiều account cùng lúc?

**Trả lời:** Dùng **AWS Organizations integration với Health**:
1. Bật `health-service-access-for-organization` từ management account
2. Xem organizational view trong Health Dashboard
3. Có thể ủy quyền cho delegated administrator account
4. Dùng EventBridge cross-account để tập trung xử lý events

---

### Q5: Phân biệt 3 loại `eventTypeCategory` trong Health?

**Trả lời:**
- **issue**: Sự cố đang xảy ra, cần phản hồi ngay (ví dụ: EC2 API degraded)
- **scheduledChange**: Thay đổi có lịch, thông báo trước để chuẩn bị (ví dụ: hardware retirement)
- **accountNotification**: Thông báo về tài khoản, không ảnh hưởng service trực tiếp (ví dụ: billing alert)

---

## Điều Hướng

| Chủ Đề                              | File                                                |
| ----------------------------------- | --------------------------------------------------- |
| Personal Health Dashboard chi tiết  | [1-personal-health.md](./1-personal-health.md)     |
| EventBridge automation chi tiết     | [2-eventbridge-integration.md](./2-eventbridge-integration.md) |
| Tổng quan management services       | [../README.md](../README.md)                        |
| Trusted Advisor (so sánh)           | [../08-trusted-advisor/README.md](../08-trusted-advisor/README.md) |
| CloudWatch (monitoring)             | [../01-cloudwatch/README.md](../01-cloudwatch/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Module:** 09 / 11 — AWS Management & Governance
