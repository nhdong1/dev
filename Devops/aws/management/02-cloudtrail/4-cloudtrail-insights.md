# CloudTrail Insights — Phát Hiện Hoạt Động API Bất Thường

> **CloudTrail Insights** (Nhận Thức CloudTrail) dùng machine learning (học máy) để liên tục phân tích API activity và tạo ra Insights Events khi phát hiện hoạt động bất thường so với baseline thông thường — công cụ phát hiện mối đe dọa (threat detection) tự động mà không cần viết rule thủ công.

---

## 📚 Mục Lục

1. [CloudTrail Insights Là Gì?](#cloudtrail-insights-là-gì)
2. [Hai Loại Insights Phát Hiện](#hai-loại-insights-phát-hiện)
3. [Cơ Chế Baseline & Anomaly Detection](#cơ-chế-baseline--anomaly-detection)
4. [Cấu Trúc Insights Event](#cấu-trúc-insights-event)
5. [Bật CloudTrail Insights](#bật-cloudtrail-insights)
6. [Responding to Insights — Phản Ứng Với Cảnh Báo](#responding-to-insights--phản-ứng-với-cảnh-báo)
7. [Tích Hợp EventBridge — Tự Động Hóa Response](#tích-hợp-eventbridge--tự-động-hóa-response)
8. [CloudTrail Insights vs GuardDuty](#cloudtrail-insights-vs-guardduty)
9. [Pricing & Tối Ưu Chi Phí](#pricing--tối-ưu-chi-phí)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CloudTrail Insights Là Gì?

Trong môi trường AWS bình thường, mỗi dịch vụ và workload có **pattern API call ổn định**: CI/CD pipeline chạy vào giờ nhất định, Auto Scaling scale theo traffic, Lambda invoke theo schedule.

Khi có **sự cố bảo mật hoặc lỗi hệ thống**, pattern này bị phá vỡ:
- Kẻ tấn công có credential chạy script tạo EC2 hàng loạt
- Ransomware xóa S3 objects hàng nghìn cái/phút
- Bug trong deployment tạo loop gọi API liên tục

**CloudTrail Insights** phát hiện những gián đoạn này **tự động** — không cần biết trước ngưỡng cụ thể, vì ngưỡng được học từ lịch sử thực tế của account.

---

## Hai Loại Insights Phát Hiện

### 1. ApiCallRateInsight — Tốc Độ Gọi API Bất Thường

Phát hiện khi số lượng API write calls trong một khoảng thời gian vượt xa mức bình thường.

**Ví dụ tình huống:**

| Tình Huống                     | Baseline Bình Thường  | Khi Có Bất Thường       | Dấu Hiệu Gì              |
| ------------------------------ | --------------------- | ----------------------- | ------------------------ |
| Cryptomining sau khi key lộ    | RunInstances: 2/giờ   | RunInstances: 500/giờ   | Tấn công tài nguyên      |
| Ransomware                     | DeleteObject: 50/giờ  | DeleteObject: 50,000/giờ| Xóa dữ liệu hàng loạt   |
| Privilege escalation script    | AttachPolicy: 1/ngày  | AttachPolicy: 200/phút  | Leo thang quyền          |
| IAM enumeration (trinh sát)    | ListUsers: 5/ngày     | ListUsers: 1000/phút    | Attacker đang scan       |
| Loop bug trong Lambda          | PutItem: 100/phút     | PutItem: 100,000/phút   | Bug tạo vòng lặp vô hạn  |

### 2. ApiErrorRateInsight — Tỷ Lệ Lỗi API Bất Thường (từ 2022)

Phát hiện khi tỷ lệ lỗi (error rate) của API calls tăng đột ngột — dấu hiệu của:
- **Brute-force attack:** Kẻ tấn công thử nhiều quyền → nhiều `AccessDenied`
- **Credential misconfiguration:** App deploy với config sai → spike `InvalidClientTokenId`
- **Resource exhaustion:** Hết quota → nhiều `LimitExceeded`
- **Dependency failure:** Service B gọi Service A nhưng A lỗi → error cascade

---

## Cơ Chế Baseline & Anomaly Detection

### Xây Dựng Baseline

```
CloudTrail Insights học từ:
  - 7 ngày lịch sử gần nhất (sliding window — cửa sổ trượt)
  - Tính toán cho từng API operation (eventName) riêng lẻ
  - Tính theo giờ trong ngày và ngày trong tuần (seasonality — tính mùa vụ)

Baseline không phải con số cố định — nó liên tục cập nhật:
  Ví dụ: RunInstances
    Thứ 2–6: Baseline 10/giờ (giờ hành chính cao điểm)
    Thứ 7–8: Baseline 1/giờ (ít deploy cuối tuần)
    2–5 giờ sáng: Baseline 0.5/giờ
```

### Phát Hiện Bất Thường

```
┌─────────────────────────────────────────────────────────────┐
│                ANOMALY DETECTION LOGIC                       │
│                                                             │
│  Current Rate: 500 RunInstances/hour                        │
│  Baseline:     10 RunInstances/hour                         │
│                                                             │
│  Deviation = (500 - 10) / 10 = 4900% deviation             │
│                                                             │
│  Threshold: Statistically significant (ML-determined)       │
│                                                             │
│  Result: ⚠️ INSIGHT EVENT GENERATED                        │
│                                                             │
│  Insight State: "Start" → (activity continues) → "End"     │
└─────────────────────────────────────────────────────────────┘
```

### Insight Event Lifecycle (Vòng Đời Sự Kiện)

```
Thời điểm bất thường bắt đầu
        │
        ▼ (sau ~15-30 phút)
  Insights Event: State = "Start"
  (ghi vào S3 + CloudWatch Logs + EventBridge)
        │
        │ (bất thường tiếp tục)
        │
        ▼ (sau khi rate trở về bình thường)
  Insights Event: State = "End"
  (tóm tắt duration và peak rate)
```

---

## Cấu Trúc Insights Event

```json
{
  "eventVersion": "1.08",
  "eventTime": "2026-05-17T04:15:00Z",
  "eventType": "AwsCloudTrailInsight",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "RunInstances",
  "awsRegion": "us-east-1",
  "insightDetails": {
    "state": "Start",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "RunInstances",
    "insightType": "ApiCallRateInsight",
    "insightContext": {
      "statistics": {
        "baseline": {
          "average": 5.2,
          "max": 12.0,
          "min": 0.0
        },
        "insight": {
          "average": 203.7,
          "max": 503.0,
          "min": 182.0
        },
        "insightDuration": 15,
        "baselineDuration": 10080
      },
      "attributions": [
        {
          "attribute": "userIdentityArn",
          "insight": [
            {
              "value": "arn:aws:iam::123456789012:user/compromised-user",
              "average": 200.1
            }
          ],
          "baseline": [
            {
              "value": "arn:aws:iam::123456789012:role/AutoScalingRole",
              "average": 5.2
            }
          ]
        }
      ]
    }
  }
}
```

### Trường Quan Trọng Trong Insights Event

| Trường                         | Ý Nghĩa                                              |
| ------------------------------ | ---------------------------------------------------- |
| `insightDetails.state`         | `Start` = bắt đầu bất thường, `End` = kết thúc      |
| `insightType`                  | `ApiCallRateInsight` hoặc `ApiErrorRateInsight`      |
| `statistics.baseline.average`  | Rate bình thường (calls/phút)                        |
| `statistics.insight.average`   | Rate bất thường hiện tại                             |
| `insightDuration`              | Số phút bất thường kéo dài                           |
| `baselineDuration`             | Số phút dữ liệu baseline (thường 10080 = 7 ngày)    |
| `attributions`                 | **Ai** gây ra bất thường (IAM identity)              |

### Trường `attributions` — Ai Gây Ra Bất Thường?

`attributions` là thông tin quan trọng nhất: CloudTrail so sánh ai thực hiện API call trong period bất thường vs baseline, để xác định actor gây ra sự kiện:

```json
"attributions": [
  {
    "attribute": "userIdentityArn",
    "insight": [
      {
        "value": "arn:aws:iam::123456789012:user/hacker-account",
        "average": 490.5
      }
    ],
    "baseline": [
      {
        "value": "arn:aws:iam::123456789012:role/LegitimateRole",
        "average": 5.1
      }
    ]
  }
]
```

Đây là bằng chứng: `hacker-account` gọi `RunInstances` 490 lần/phút, trong khi bình thường `LegitimateRole` chỉ gọi 5 lần/phút.

---

## Bật CloudTrail Insights

### Qua AWS Console

```
CloudTrail → Trails → Chọn Trail → Edit
→ Insights → Check "API call rate" và/hoặc "API error rate"
→ Save
```

### Qua AWS CLI

```bash
# Bật cả hai loại Insights
aws cloudtrail put-insight-selectors \
  --trail-name my-trail \
  --insight-selectors \
    '[{"InsightType": "ApiCallRateInsight"},
      {"InsightType": "ApiErrorRateInsight"}]'

# Kiểm tra cấu hình
aws cloudtrail get-insight-selectors --trail-name my-trail
```

### Qua CloudFormation

```yaml
Trail:
  Type: AWS::CloudTrail::Trail
  Properties:
    TrailName: my-trail
    S3BucketName: !Ref CloudTrailBucket
    IsLogging: true
    IsMultiRegionTrail: true
    InsightSelectors:
      - InsightType: ApiCallRateInsight
      - InsightType: ApiErrorRateInsight
```

### Nơi Insights Events Được Lưu

Insights Events được ghi vào thư mục riêng trong cùng S3 bucket:

```
s3://my-cloudtrail-bucket/
├── AWSLogs/{AccountId}/CloudTrail/{Region}/...     ← Regular events
└── AWSLogs/{AccountId}/CloudTrail-Insight/{Region}/...  ← Insights events
```

---

## Responding to Insights — Phản Ứng Với Cảnh Báo

### Quy Trình Điều Tra Khi Nhận Insights Alert

```
Bước 1: Xác nhận bất thường có thực không
         → Xem baseline vs insight rate
         → Kiểm tra "attributions" — ai gây ra?
         → Có scheduled job, deployment, migration hợp lệ không?

Bước 2: Xác định scope (phạm vi)
         → Event xảy ra ở region nào?
         → Bắt đầu lúc nào? Kéo dài bao lâu?
         → Bao nhiêu resources bị tác động?

Bước 3: Query Chi Tiết Events Trong Period Bất Thường
         → Dùng Athena hoặc CloudWatch Logs Insights
         → Lọc theo eventName + timeWindow

Bước 4: Quyết định response
         → Nếu hợp lệ (deployment, migration): document và close
         → Nếu nghi ngờ tấn công: escalate, isolate, investigate
```

### Query Athena Khi Nhận Insight Alert

```sql
-- Tìm chi tiết tất cả RunInstances trong window bất thường
SELECT
    eventTime,
    userIdentity.arn AS actor,
    userIdentity.accountId AS account,
    awsRegion,
    sourceIPAddress,
    requestParameters,
    responseElements
FROM cloudtrail_logs
WHERE eventName = 'RunInstances'
  AND eventTime BETWEEN '2026-05-17T04:00:00Z' AND '2026-05-17T05:00:00Z'
ORDER BY eventTime;
```

```sql
-- Tìm xem instances nào đã được tạo (để terminate nếu cần)
SELECT
    json_extract_scalar(responseElements, '$.instancesSet.items[0].instanceId') AS instance_id,
    eventTime,
    userIdentity.arn AS created_by,
    awsRegion
FROM cloudtrail_logs
WHERE eventName = 'RunInstances'
  AND eventTime > '2026-05-17T04:00:00Z';
```

---

## Tích Hợp EventBridge — Tự Động Hóa Response

CloudTrail Insights Events tự động gửi đến **Amazon EventBridge** — cho phép trigger Lambda để phản ứng tự động:

### EventBridge Rule Cho Insights Events

```json
{
  "source": ["aws.cloudtrail"],
  "detail-type": ["AWS Insight via CloudTrail"],
  "detail": {
    "insightDetails": {
      "state": ["Start"],
      "insightType": ["ApiCallRateInsight"]
    }
  }
}
```

### Lambda Handler — Automated Response

```python
import boto3
import json

def handler(event, context):
    detail = event['detail']
    insight_details = detail['insightDetails']

    event_name = insight_details['eventName']
    event_source = insight_details['eventSource']
    state = insight_details['state']
    attributions = insight_details['insightContext'].get('attributions', [])

    # Tìm actor gây ra bất thường
    suspicious_actors = []
    for attr in attributions:
        if attr['attribute'] == 'userIdentityArn':
            for item in attr.get('insight', []):
                suspicious_actors.append(item['value'])

    print(f"Insight: {event_name} from {event_source} - State: {state}")
    print(f"Suspicious actors: {suspicious_actors}")

    # Ví dụ: Nếu là RunInstances spike → kiểm tra và terminate instances lạ
    if event_name == 'RunInstances' and state == 'Start':
        handle_ec2_spike(suspicious_actors)

    # Gửi alert cho Security team
    send_security_alert(event_name, suspicious_actors, insight_details)

def handle_ec2_spike(actors):
    ec2 = boto3.client('ec2')
    iam = boto3.client('iam')

    for actor_arn in actors:
        # Option 1: Attach DenyAll policy tạm thời để chặn actor
        if ':user/' in actor_arn:
            user_name = actor_arn.split('/')[-1]
            iam.put_user_policy(
                UserName=user_name,
                PolicyName='EmergencyDenyAll',
                PolicyDocument=json.dumps({
                    "Version": "2012-10-17",
                    "Statement": [{
                        "Effect": "Deny",
                        "Action": "*",
                        "Resource": "*"
                    }]
                })
            )
            print(f"QUARANTINED user: {user_name}")

def send_security_alert(event_name, actors, details):
    sns = boto3.client('sns')
    message = f"""
🚨 SECURITY ALERT: CloudTrail Insights Detected Anomaly

Event: {event_name}
Suspicious Actors: {actors}
Baseline Rate: {details['insightContext']['statistics']['baseline']['average']:.1f}/min
Anomaly Rate: {details['insightContext']['statistics']['insight']['average']:.1f}/min

Immediate investigation required!
    """
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
        Subject=f'CloudTrail Insights Alert: {event_name} Spike',
        Message=message
    )
```

### Kiến Trúc Automated Security Response

```
CloudTrail Insights Event
        │
        ▼
Amazon EventBridge
  Rule: InsightType = ApiCallRateInsight
        │
        ├──→ Lambda: Quarantine suspicious IAM user
        │
        ├──→ Lambda: Create incident ticket (Jira/PagerDuty)
        │
        ├──→ SNS → PagerDuty/Slack/Email (alert security team)
        │
        └──→ Lambda: Snapshot EC2 instances (forensics evidence)
```

---

## CloudTrail Insights vs GuardDuty

Hai dịch vụ hay bị nhầm lẫn vì cùng phát hiện bất thường:

| Tiêu Chí                   | CloudTrail Insights                   | Amazon GuardDuty                           |
| -------------------------- | ------------------------------------- | ------------------------------------------ |
| **Focus**                  | API call rate/error rate bất thường   | Threat intelligence, behavior analytics    |
| **Data source**            | CloudTrail Management Events          | CloudTrail + VPC Flow Logs + DNS logs       |
| **Phát hiện**              | Statistical anomaly (tự học baseline) | Known threat patterns + ML + threat intel  |
| **Coverage**               | AWS API activity                      | AWS API + network + DNS                    |
| **Remediation**            | Không có sẵn (tự implement)           | Tích hợp với Security Hub, Lambda          |
| **Thời gian phát hiện**    | 15–30 phút sau bất thường             | Near real-time (phút)                      |
| **False positives**        | Có thể cao (new deployments)          | Thấp hơn (nhiều context hơn)               |
| **Chi phí**                | $0.35/100k events analyzed            | Dựa trên data volume                       |
| **Best for**               | API rate anomaly detection            | Comprehensive threat detection             |

**Khuyến nghị:** Dùng **cả hai** — GuardDuty là primary threat detection, CloudTrail Insights là bổ sung để phát hiện API-level anomalies.

---

## Pricing & Tối Ưu Chi Phí

### Tính Phí Insights

```
$0.35 mỗi 100,000 write management events được phân tích
(KHÔNG phải mỗi Insights Event tạo ra — mà theo volume events được phân tích)

Ví dụ:
  Account có 10 triệu write management events/tháng
  Chi phí: 10,000,000 / 100,000 × $0.35 = $35/tháng
```

### Khi Nào Nên Bật Insights?

```
NÊN BẬT khi:
  ✅ Có security team theo dõi và respond
  ✅ Compliance yêu cầu threat detection (PCI-DSS, SOC2)
  ✅ Production environment có high-value data
  ✅ Đã tích hợp EventBridge → Lambda để automate response

KHÔNG CẦN ngay khi:
  ❌ Startup nhỏ, chưa có security team
  ❌ Dev/test environment
  ❌ Budget eo hẹp (GuardDuty cover nhiều hơn và thường rẻ hơn)
  ❌ Chưa có plan để respond tới alert
```

---

## Câu Hỏi Phỏng Vấn

### Q1: CloudTrail Insights học baseline trong bao lâu?

**7 ngày** (168 giờ) — cửa sổ trượt (sliding window). Điều này có nghĩa:
- Nếu bật Insights cho Trail mới, phải chờ 7 ngày trước khi có đủ data để phát hiện bất thường
- Baseline liên tục cập nhật — sau 7 ngày ban đầu, mỗi giờ trôi qua, giờ cũ nhất bị loại và giờ mới thêm vào

---

### Q2: Insights Event xuất hiện ngay lập tức không?

**Không.** Latency thường là 15–30 phút từ khi bất thường bắt đầu đến khi Insights Event được tạo. Đây là đánh đổi giữa độ chính xác (tránh false positive với spike ngắn) và tốc độ phát hiện.

Nếu cần phát hiện **realtime hơn**, dùng CloudWatch Alarms với Metric Filters trên specific event names.

---

### Q3: Làm thế nào giảm false positives từ Insights?

**False positives** thường xảy ra khi:
- Có deployment lớn (hàng trăm EC2 scale out đột ngột)
- Database migration tạo hàng nghìn PutItem
- Load test trên môi trường production

**Cách giảm false positive:**
1. Implement alert routing qua Lambda — kiểm tra maintenance window calendar trước khi page security team
2. Tag resources được tạo bởi automation để phân biệt
3. Lọc attributions — nếu actor là known automation role, lower severity
4. Bật Insights trên production Trail riêng, không bao gồm staging events

---

### Q4: Nếu kẻ tấn công biết về Insights và cố ý gọi API chậm để không trigger, có phát hiện được không?

**Đây là limitation của CloudTrail Insights.** Kẻ tấn công thực hiện slow-and-low attack (tấn công chậm, ổn định) có thể không vượt ngưỡng rate. Đây là lý do cần **kết hợp nhiều lớp bảo mật**:

- CloudTrail Insights: API rate anomaly
- GuardDuty: Threat intelligence, known bad IPs, behavior patterns
- AWS Config: Detect configuration changes
- IAM Access Analyzer: Phân tích quyền được cấp
- Security Hub: Tổng hợp tất cả findings

Không dịch vụ nào là "silver bullet" — defense in depth (phòng thủ theo chiều sâu) là nguyên tắc quan trọng nhất.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [3-organization-trail.md](./3-organization-trail.md) | [5-forensics-investigation.md](./5-forensics-investigation.md)
