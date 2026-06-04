# Top 30 Câu Hỏi Phỏng Vấn — AWS Management & Governance

> Mỗi câu hỏi có: mức độ khó, đáp án chi tiết, điểm cộng gây ấn tượng và lỗi thường gặp. Học theo thứ tự từ Q1 (dễ nhất) đến Q30 (khó nhất).

---

## Phân Loại Theo Chủ Đề

| Nhóm                           | Câu Hỏi | Mức Độ    |
| ------------------------------ | ------- | --------- |
| CloudWatch & Observability     | Q1–Q6   | Cơ bản    |
| CloudTrail & Audit             | Q7–Q11  | Cơ bản    |
| AWS Config & Compliance        | Q12–Q15 | Trung cấp |
| Systems Manager (SSM)          | Q16–Q18 | Trung cấp |
| CloudFormation & IaC           | Q19–Q22 | Trung cấp |
| Organizations & Multi-Account  | Q23–Q26 | Nâng cao  |
| Control Tower & Landing Zone   | Q27–Q28 | Nâng cao  |
| Cost Governance & FinOps       | Q29–Q30 | Nâng cao  |

---

## PHẦN 1: CLOUDWATCH & OBSERVABILITY

---

### Q1. CloudWatch Metrics vs CloudWatch Logs khác nhau như thế nào?

**Mức độ:** ⭐ Cơ bản | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**CloudWatch Metrics** (Chỉ Số CloudWatch) là dữ liệu dạng số theo thời gian — numeric time-series data. Ví dụ: CPU = 75%, Request Count = 1,200/phút, Error Rate = 0.5%. Metrics được tổng hợp theo period (chu kỳ) và dimension (chiều phân loại).

**CloudWatch Logs** (Nhật Ký CloudWatch) là dữ liệu log text thô từ ứng dụng, OS, Lambda, VPC Flow Logs. Logs lưu trong Log Groups → Log Streams, có thể query bằng Logs Insights.

| Khía Cạnh          | Metrics                      | Logs                          |
| ------------------ | ---------------------------- | ----------------------------- |
| Dạng dữ liệu       | Số (numeric)                 | Text (semi-structured)        |
| Dùng để            | Alerting, dashboards, scaling | Debug, audit, forensics       |
| Query              | Metric Math, GetMetricData   | Logs Insights (SQL-like)      |
| Chi phí giữ lâu    | Standard: 15 tháng           | Tự cấu hình retention 1–3653 ngày |
| Granularity nhỏ nhất | 1 giây (high-resolution)   | Theo từng log event           |

**Điểm Cộng:** Đề cập rằng từ Logs có thể tạo **Metric Filter** (Bộ Lọc Chỉ Số) để convert log pattern thành Metric — ví dụ: đếm số lần `ERROR` xuất hiện → tạo metric `ErrorCount`.

**Lỗi Thường Gặp:** Nghĩ rằng Metrics và Logs là thay thế nhau — thực ra bổ sung cho nhau: Metrics cho biết "có vấn đề", Logs cho biết "vấn đề là gì".

---

### Q2. CloudWatch Alarm có những state nào và ý nghĩa là gì?

**Mức độ:** ⭐ Cơ bản | **Hay hỏi:** ✅✅✅

**Đáp Án:**

CloudWatch Alarm có 3 states (trạng thái):

**1. OK** — Metric đang trong ngưỡng bình thường, không có vấn đề.

**2. ALARM** — Metric vượt ngưỡng (threshold) trong số lần đánh giá (evaluation periods) được cấu hình. Actions (hành động) được trigger: SNS notification, Auto Scaling, EC2 action...

**3. INSUFFICIENT_DATA** — Không đủ dữ liệu để đánh giá. Xảy ra khi: instance mới khởi động, metric chưa được báo cáo, hoặc CloudWatch chưa nhận đủ data points. Có thể cấu hình treat as OK hoặc ALARM.

**Cấu hình quan trọng:**
```
Period: 60 giây (độ granularity đánh giá)
EvaluationPeriods: 3 (số period liên tiếp phải vượt ngưỡng)
DatapointsToAlarm: 2 (trong 3 periods, cần ít nhất 2 điểm vượt ngưỡng)
→ Giúp tránh false alarm từ spike ngắn
```

**Điểm Cộng:** Nhắc đến **Composite Alarm** (Cảnh Báo Tổng Hợp) — kết hợp nhiều alarm với AND/OR logic. Ví dụ: ALARM khi CPU > 80% VÀ Memory > 90% — tránh alert khi chỉ có một metric vượt ngưỡng tạm thời.

---

### Q3. CloudWatch Logs Insights dùng để làm gì? Viết query cơ bản.

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

CloudWatch Logs Insights (Phân Tích Nhật Ký Chuyên Sâu) là công cụ query tương tác để phân tích log trong CloudWatch theo cú pháp giống SQL.

**Các lệnh chính:**
- `fields` — chọn fields hiển thị
- `filter` — lọc theo điều kiện
- `stats` — tổng hợp (count, sum, avg, min, max)
- `sort` — sắp xếp
- `limit` — giới hạn kết quả

**Ví dụ Query:**

```
# Tìm tất cả ERROR logs trong 1 giờ qua
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

# Đếm errors theo loại
fields @message
| filter @message like /ERROR/
| parse @message "ERROR: *" as errorType
| stats count(*) as errorCount by errorType
| sort errorCount desc

# Tìm Lambda invocations chậm nhất
fields @timestamp, @duration, @requestId
| filter @type = "REPORT"
| sort @duration desc
| limit 20

# VPC Flow Logs — top IPs rejected
fields srcAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) as rejectCount by srcAddr
| sort rejectCount desc
| limit 10
```

**Điểm Cộng:** Đề cập có thể query **nhiều Log Groups cùng lúc** (cross-log-group queries) và **lưu query** để reuse.

---

### Q4. Thiết kế hệ thống cảnh báo cho production web application. Bạn sẽ alarm gì?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

Dựa theo **4 Golden Signals** (4 Tín Hiệu Vàng) của Site Reliability Engineering:

**1. Latency (Độ Trễ):**
- ALB — TargetResponseTime P95 > 2 giây → ALARM
- ALB — TargetResponseTime P99 > 5 giây → ALARM (critical)

**2. Traffic (Lưu Lượng):**
- ALB — RequestCount giảm 50% so với baseline → ALARM (có thể routing issue)
- Metric Math: Error Rate = (5xx Count / Total Request Count) × 100

**3. Errors (Lỗi):**
- ALB — HTTPCode_Target_5XX_Count > 50 trong 5 phút → ALARM
- Lambda — Errors > 5 trong 5 phút → ALARM
- CloudWatch Metric Filter: `/ERROR/` pattern → ErrorRate metric

**4. Saturation (Bão Hòa):**
- EC2 — CPUUtilization > 80% trong 5 phút (2 evaluation periods) → ALARM
- RDS — FreeStorageSpace < 10GB → ALARM
- SQS — ApproximateNumberOfMessagesVisible > 10,000 → ALARM (queue backlog)

**Bổ sung:**
- Composite Alarm: `(5xx_alarm OR latency_alarm) AND traffic_alarm NOT zero`
- Dashboard: Traffic Light view — xanh/vàng/đỏ theo alarm state

**Điểm Cộng:** Đề cập **CloudWatch Container Insights** cho ECS/EKS và **Application Insights** cho automatic problem detection.

---

### Q5. CloudWatch Container Insights là gì? Khi nào dùng?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅

**Đáp Án:**

CloudWatch Container Insights (Phân Tích Chuyên Sâu Container) là tính năng thu thập, tổng hợp và tóm tắt metrics và logs từ các workloads container: Amazon ECS, Amazon EKS, Kubernetes trên EC2, và AWS Fargate.

**Dữ liệu thu thập:**
- CPU và Memory sử dụng theo cluster → service → task → container
- Network I/O, Disk I/O
- Số lượng pod, task đang chạy/pending/stopped
- Container restart count

**Lợi ích:**
- Không cần cài đặt agent tùy chỉnh (built-in với ECS, cần cấu hình cho EKS)
- Dashboard sẵn có trong CloudWatch Console
- Có thể tạo alarm theo service-level metrics

**Bật Container Insights:**
```bash
# ECS — bật khi tạo cluster
aws ecs put-account-setting \
  --name containerInsights \
  --value enabled

# EKS — cài CloudWatch agent qua DaemonSet
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability
```

**Dùng khi:** Vận hành ECS/EKS ở production, cần visibility vào container performance mà không muốn quản lý Prometheus/Grafana.

---

### Q6. CloudWatch Agent là gì và khi nào cần cài?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

CloudWatch Agent (Tác Nhân CloudWatch) là phần mềm cài trên EC2 instance (hoặc on-premises server) để thu thập metrics và logs vượt ra ngoài những gì AWS tự động báo cáo.

**Tại sao cần:**
- EC2 mặc định chỉ báo cáo CPU, Network, Disk I/O tới CloudWatch
- Không có **Memory utilization** (sử dụng bộ nhớ) — AWS không thể đọc từ hypervisor
- Không có **Disk usage** (% đĩa đầy) — chỉ có Disk I/O
- Không có **custom application logs** nếu không cài agent

**Những gì CloudWatch Agent thêm vào:**
- Memory (RAM) usage percentage
- Swap usage
- Disk utilization (% used)
- Custom log files: `/var/log/app/*.log`, Windows Event Logs
- Process-level metrics

**Cài đặt qua SSM:**
```bash
# Dùng SSM Run Command để cài hàng loạt
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}' \
  --targets '[{"Key":"tag:Environment","Values":["production"]}]'
```

**Lưu ý:** EC2 cần IAM role có policy `CloudWatchAgentServerPolicy` để gửi data.

---

## PHẦN 2: CLOUDTRAIL & AUDIT

---

### Q7. CloudTrail là gì và ghi lại những gì?

**Mức độ:** ⭐ Cơ bản | **Hay hỏi:** ✅✅✅

**Đáp Án:**

CloudTrail (Dấu Vết Đám Mây) là dịch vụ kiểm toán (audit) ghi lại tất cả **API calls** (lời gọi API) trên AWS account — ai làm gì, khi nào, từ đâu, kết quả thế nào.

**3 loại events:**

**1. Management Events** (Sự Kiện Quản Lý) — Mặc định bật, miễn phí cho Trail đầu tiên:
- Control plane operations: `CreateBucket`, `DeleteInstance`, `AttachRolePolicy`
- Thay đổi cấu hình tài nguyên

**2. Data Events** (Sự Kiện Dữ Liệu) — Phải bật thêm, tính phí:
- S3: `GetObject`, `PutObject`, `DeleteObject`
- Lambda: `Invoke`
- DynamoDB: `GetItem`, `PutItem`

**3. CloudTrail Insights Events** (Sự Kiện Phân Tích) — Tính phí:
- Phát hiện hoạt động API bất thường (anomaly detection)
- So sánh với baseline và alert khi có deviation

**Mỗi event ghi:**
- `userIdentity` — ai thực hiện (IAM user, role, service)
- `eventTime` — thời điểm
- `sourceIPAddress` — IP nguồn
- `eventName` — tên API call
- `requestParameters` / `responseElements` — chi tiết request/response

---

### Q8. Làm thế nào phát hiện ai đã xóa S3 bucket?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**Bước 1: Dùng CloudTrail Event History** (nhanh nhất, 90 ngày)

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
    AttributeKey=EventName,AttributeValue=DeleteBucket \
  --start-time 2026-05-01 \
  --end-time 2026-05-17 \
  --query 'Events[*].{User:Username,Time:EventTime,Bucket:Resources[0].ResourceName}'
```

**Bước 2: Query Athena** (nếu cần lịch sử > 90 ngày)

```sql
SELECT
  useridentity.arn as who,
  eventtime as when,
  requestparameters as what,
  sourceipaddress as from_where
FROM cloudtrail_logs
WHERE eventname = 'DeleteBucket'
  AND eventtime BETWEEN '2026-05-01' AND '2026-05-17'
ORDER BY eventtime DESC;
```

**Bước 3: Filter trong Console**

CloudTrail Console → Event History → Filter: Event name = `DeleteBucket`

**Điểm Cộng:** Đề cập thiết lập **EventBridge rule** để alert real-time khi `DeleteBucket` xảy ra, thay vì phải điều tra sau khi sự việc đã xảy ra.

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["DeleteBucket"]
  }
}
```

---

### Q9. Organization Trail là gì? Tại sao nên dùng thay vì trail riêng mỗi account?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

**Organization Trail** (Dấu Vết Tổ Chức) là CloudTrail trail được tạo từ **Management Account** của AWS Organizations và tự động áp dụng cho **tất cả member accounts** trong organization, kể cả accounts được tạo mới sau này.

**Lợi ích so với trail riêng từng account:**

| Khía Cạnh               | Trail Riêng Mỗi Account                   | Organization Trail                          |
| ----------------------- | ----------------------------------------- | ------------------------------------------- |
| Quản lý                 | N account = N trail configuration        | 1 trail áp dụng cho tất cả                 |
| Account mới             | Phải nhớ bật trail thủ công              | Tự động áp dụng ngay khi tạo account        |
| Tắt trail               | Developer có thể tắt nếu có quyền        | Member accounts không thể tắt              |
| Centralized logging     | Phải cấu hình S3 cross-account riêng     | Tất cả log tập trung vào 1 S3 bucket       |
| Phí                     | Management Events: miễn phí trail đầu   | Chỉ tính phí data events trên members      |

**Tạo Organization Trail:**
```bash
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name centralized-audit-logs \
  --is-organization-trail \
  --is-multi-region-trail
```

**Điểm Cộng:** Nhắc rằng Organization Trail nên lưu vào **Log Archive account** (tài khoản lưu trữ nhật ký) — một account riêng trong Security OU mà chỉ security team có quyền đọc. Đây là pattern chuẩn trong AWS Control Tower.

---

### Q10. CloudTrail Insights là gì? Khác gì với Alarms thông thường?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅

**Đáp Án:**

**CloudTrail Insights** (Phân Tích Chuyên Sâu CloudTrail) sử dụng machine learning (học máy) để phát hiện hoạt động API bất thường (anomalous API activity) bằng cách so sánh với baseline (đường cơ sở) lịch sử.

**Cách hoạt động:**
1. CloudTrail học baseline: số lượng API calls "bình thường" trong giờ nào, ngày nào
2. Khi phát hiện spike bất thường (ví dụ: `CreateSecurityGroup` được gọi 200 lần trong 10 phút thay vì 5 lần bình thường) → tạo Insights Event
3. Insights Event được ghi vào S3 riêng và có thể route qua EventBridge

**Khác với CloudWatch Alarms:**
- CloudWatch Alarm: so sánh metric với **threshold cố định** (CPU > 80%)
- CloudTrail Insights: phát hiện **deviation từ baseline động** — phù hợp cho API activity patterns thay đổi theo thời gian

**Khi nào cần:**
- Phát hiện bất thường bảo mật: ai đó đột ngột tạo hàng trăm IAM roles
- Phát hiện lỗi automation: script bị loop gọi API hàng nghìn lần
- Compliance: chứng minh có anomaly detection cho PCI-DSS / SOC2

**Lưu ý:** CloudTrail Insights tính phí thêm ngoài trail thông thường.

---

### Q11. CloudTrail vs AWS Config — khi nào dùng cái nào?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

Đây là câu hỏi rất hay gặp. Nhớ theo công thức: **"Hành động" vs "Trạng Thái"**.

| Khía Cạnh                   | CloudTrail                              | AWS Config                                 |
| --------------------------- | --------------------------------------- | ------------------------------------------ |
| **Câu hỏi**                 | "Ai đã làm gì và khi nào?"             | "Cấu hình hiện tại có đúng không?"         |
| **Dữ liệu**                 | API call events (hành động)             | Resource configuration state (trạng thái) |
| **Nguồn**                   | AWS Control Plane                       | Tất cả tài nguyên AWS                     |
| **Thời gian**               | Theo sự kiện xảy ra                     | Theo thay đổi cấu hình                    |
| **Compliance check**        | Không tự kiểm tra compliance            | Có Config Rules tự động kiểm tra           |
| **Remediation**             | Không có native                          | Có Auto-Remediation qua SSM               |
| **Lưu trữ chính**          | S3                                       | S3                                         |

**Ví dụ phân biệt:**
- "Security group vừa thay đổi inbound rule lúc 2 giờ sáng, ai làm?" → **CloudTrail**
- "Security group nào đang cho phép port 22 từ 0.0.0.0/0 ngay lúc này?" → **AWS Config**
- "Security group thay đổi khi nào và từ cấu hình nào sang cấu hình nào?" → **AWS Config** (timeline)
- "IAM role nào đã bị attach policy mới hôm nay?" → **CloudTrail**

**Thực tế thường dùng cả hai:**

```
Config Rule phát hiện violation → Config ghi vào S3
                                 → EventBridge trigger
                                 → Lambda query CloudTrail: ai gây ra?
                                 → Kết hợp: biết CẢ "cấu hình sai" VÀ "ai làm sai"
```

---

## PHẦN 3: AWS CONFIG & COMPLIANCE

---

### Q12. AWS Config Rules là gì? Phân biệt Managed Rules và Custom Rules.

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

**Config Rules** (Quy Tắc Config) là các quy tắc tự động đánh giá cấu hình tài nguyên AWS có tuân thủ policy không.

**Managed Rules** (Quy Tắc Được Quản Lý) — AWS cung cấp sẵn:
- Hơn 200 rules có sẵn
- Không cần viết code
- Ví dụ: `s3-bucket-versioning-enabled`, `rds-multi-az-support`, `restricted-ssh`

**Custom Rules** (Quy Tắc Tùy Chỉnh) — Tự viết:
- **Lambda-based** (cũ): Viết Lambda function đánh giá và gọi `put-evaluations` API
- **Guard-based** (mới, không cần Lambda): Dùng AWS CloudFormation Guard DSL

**Custom Lambda Rule — Cấu trúc:**
```python
def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    config_item = invoking_event['configurationItem']
    
    # Logic đánh giá
    if config_item['resourceType'] == 'AWS::EC2::Instance':
        tags = {t['key']: t['value'] for t in config_item.get('tags', [])}
        compliance = 'COMPLIANT' if 'Owner' in tags else 'NON_COMPLIANT'
    else:
        compliance = 'NOT_APPLICABLE'
    
    # Báo cáo kết quả
    config_client.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': config_item['resourceType'],
            'ComplianceResourceId': config_item['resourceId'],
            'ComplianceType': compliance,
            'OrderingTimestamp': config_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
```

**Trigger types:**
- **Configuration change** — Đánh giá khi resource thay đổi
- **Periodic** — Đánh giá theo lịch (mỗi 1/3/6/12/24 giờ)

---

### Q13. Config Remediation Actions hoạt động thế nào?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

**Remediation Actions** (Hành Động Khắc Phục) là khả năng AWS Config tự động (hoặc bán tự động) sửa tài nguyên non-compliant.

**Hai chế độ:**
1. **Manual Remediation** — Người phải click "Remediate" trong Console
2. **Automatic Remediation** — Tự động trigger khi phát hiện non-compliant

**Cơ chế hoạt động:**

```
Config Rule → NON_COMPLIANT → SSM Automation Document → Fix resource
                           → (Optional: SNS notification)
```

**SSM Automation Documents sẵn có:**
- `AWS-DisableS3BucketPublicReadWrite` — Tắt public access S3
- `AWS-EnableCWAlarm` — Bật CloudWatch Alarm
- `AWSConfigRemediation-EnableEncryptionOnSNSTopic` — Bật encryption SNS

**Custom Remediation:**
```yaml
# CloudFormation — Config Rule với Auto-remediation
RemediationConfiguration:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: !Ref S3PublicAccessRule
    TargetType: SSM_DOCUMENT
    TargetId: AWS-DisableS3BucketPublicReadWrite
    Automatic: true
    MaximumAutomaticAttempts: 3
    RetryAttemptSeconds: 60
    Parameters:
      BucketName:
        ResourceValue:
          Value: RESOURCE_ID
```

**Lưu ý quan trọng:**
- Automatic Remediation có thể gây downtime nếu không cẩn thận — test kỹ trong staging
- Một số actions cần approval (change management) trước khi production
- Remediation chạy dưới IAM role — phải cấp quyền đủ

---

### Q14. Conformance Pack là gì? Khi nào dùng thay vì Config Rules riêng lẻ?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅

**Đáp Án:**

**Conformance Pack** (Gói Tuân Thủ) là tập hợp Config Rules và Remediation Actions đóng gói thành một đơn vị triển khai duy nhất, thường theo một compliance framework (khung tuân thủ) cụ thể.

**Tại sao cần:**
- CIS AWS Foundations Benchmark có 40+ rules — tạo thủ công tốn thời gian
- PCI-DSS, NIST, HIPAA cũng có bộ rules riêng
- Conformance Pack giúp deploy tất cả cùng lúc, track compliance theo framework

**Pre-built Conformance Packs của AWS:**
- `Operational-Best-Practices-for-CIS-AWS-v1.4-Level1`
- `Operational-Best-Practices-for-PCI-DSS`
- `Operational-Best-Practices-for-HIPAA-Security`
- `Operational-Best-Practices-for-NIST-800-53-rev-5`

**Triển khai:**
```bash
aws configservice put-conformance-pack \
  --conformance-pack-name "PCI-DSS-Compliance" \
  --template-s3-uri s3://my-templates/pci-dss-conformance-pack.yaml \
  --delivery-s3-bucket my-config-bucket
```

**Dùng Conformance Pack khi:** Cần chứng minh tuân thủ theo framework chuẩn cho audit, hoặc cần deploy bộ rules nhất quán trên nhiều accounts.

---

### Q15. AWS Config Aggregator là gì?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅

**Đáp Án:**

**Config Aggregator** (Bộ Tổng Hợp Config) thu thập Config data từ nhiều accounts và nhiều regions về một account trung tâm — cho phép xem compliance toàn bộ tổ chức trong một nơi.

**Hai loại aggregator:**

1. **Individual Account Aggregator** — Tổng hợp từ danh sách accounts chỉ định cụ thể
2. **Organization Aggregator** — Tổng hợp tự động từ tất cả accounts trong AWS Organization

**Kiến trúc:**
```
Member Account A (us-east-1) ─┐
Member Account A (eu-west-1) ─┤
Member Account B (us-east-1) ─┼→ Aggregator Account → Dashboard tổng hợp
Member Account C (ap-east-1) ─┤   (thường là Security account)
...                           ─┘
```

**Lợi ích:**
- Xem tất cả non-compliant resources trên toàn tổ chức từ một nơi
- Không cần login vào từng account để kiểm tra
- Tạo report compliance theo account, region, resource type

---

## PHẦN 4: SYSTEMS MANAGER (SSM)

---

### Q16. SSM Session Manager tốt hơn bastion host thế nào?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**Bastion Host** (Máy Chủ Pháo Đài) là EC2 instance trong public subnet làm relay để SSH vào instance private subnet. Đây là pattern cũ với nhiều vấn đề:

- Phải quản lý và vá lỗi bastion host
- Cần manage SSH key pairs (rủi ro mất key, key rotation)
- Port 22 mở tạo attack surface
- Khó audit ai đã làm gì trong session

**SSM Session Manager** (Trình Quản Lý Phiên SSM) giải quyết tất cả:

| Khía Cạnh               | Bastion Host                          | SSM Session Manager                     |
| ----------------------- | ------------------------------------- | --------------------------------------- |
| Port 22                 | ✅ Phải mở                            | ❌ Không cần                            |
| SSH Key pair            | ✅ Cần quản lý                        | ❌ Không cần                            |
| Internet Gateway        | ✅ Cần (bastion cần public IP)        | ❌ Không cần (dùng VPC Endpoint)        |
| Audit logs              | Partial (syslog)                      | ✅ Đầy đủ: session logs → S3/CW Logs   |
| IAM control             | Hạn chế (SSH key level)              | ✅ Granular IAM policy                  |
| Quản lý infrastructure  | Phải patch, monitor bastion           | Serverless — không cần quản lý          |
| Multi-region            | Phải có bastion mỗi region            | SSM Agent là đủ                         |

**Điều kiện để dùng Session Manager:**
1. EC2 phải có SSM Agent (Amazon Linux 2 và Windows đã có sẵn)
2. EC2 cần IAM Instance Profile với `AmazonSSMManagedInstanceCore` policy
3. SSM Endpoints hoặc Internet access để reach SSM service

**Điểm Cộng:** Đề cập Session Manager hỗ trợ **Port Forwarding** (Chuyển Tiếp Cổng) — tunnel port từ laptop vào private instance mà không cần VPN.

---

### Q17. SSM Parameter Store vs AWS Secrets Manager — khi nào dùng cái nào?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**Dùng Parameter Store khi:**
- Config đơn giản: DB_HOST, APP_VERSION, FEATURE_FLAGS
- Không cần rotation tự động
- Muốn tiết kiệm chi phí (Standard tier miễn phí)
- Cần phân cấp `/app/env/key` rõ ràng
- Nhiều config variables nhỏ (hàng trăm parameters)

**Dùng Secrets Manager khi:**
- Database credentials cần rotation tự động (RDS, Redshift, DocumentDB)
- API keys của third-party service cần lifecycle management
- Cross-account secret sharing
- Cần audit trail chi tiết hơn cho compliance (HIPAA, PCI)

**Tóm tắt quyết định nhanh:**
```
Cần auto-rotation?
  Có → Secrets Manager
  Không →
    Có nhiều params (> 20)?
      Có → Parameter Store (rẻ hơn)
      Không →
        Cần cross-account?
          Có → Secrets Manager
          Không → Parameter Store
```

**Chi phí ví dụ:**
- 100 secrets trong Secrets Manager = $40/tháng
- 100 SecureString parameters trong Parameter Store Advanced = $5/tháng

---

### Q18. SSM Run Command vs SSM Automation — khác nhau gì?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅

**Đáp Án:**

**SSM Run Command** (Chạy Lệnh SSM):
- Thực thi **single command/script** trên nhiều EC2 instances
- Simple, immediate execution
- Không có workflow, không có condition/loop
- Ví dụ: restart nginx trên 50 instances, collect disk usage report

```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --parameters '{"commands":["systemctl restart nginx"]}' \
  --targets '[{"Key":"tag:App","Values":["web-server"]}]'
```

**SSM Automation** (Tự Động Hóa SSM):
- Thực thi **multi-step workflow** (runbook) phức tạp
- Có điều kiện (conditional steps), approval gates, error handling
- Có thể tác động đến AWS resources (không chỉ EC2 instances)
- Ví dụ: AMI baking workflow, automated patching với pre/post checks, disaster recovery

```
Automation Document Example (Patch Workflow):
Step 1: Create AMI backup
Step 2: Stop instances
Step 3: Apply patches (Run Command)
Step 4: Start instances  
Step 5: Verify health check
Step 6: (On failure) Restore from AMI
```

**Điểm Cộng:** SSM Automation tích hợp tốt với **AWS Config Remediation** — Config phát hiện vi phạm → trigger Automation runbook để tự fix.

---

## PHẦN 5: CLOUDFORMATION & IAC

---

### Q19. CloudFormation Stack vs StackSet — khác nhau gì?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**CloudFormation Stack** (Ngăn Xếp CloudFormation):
- Tập hợp tài nguyên AWS được tạo từ một template và quản lý cùng nhau
- Deploy vào **một account + một region**
- Vòng đời: create → update → delete

**CloudFormation StackSet** (Bộ Ngăn Xếp CloudFormation):
- Mở rộng Stack để deploy đồng thời trên **nhiều accounts và/hoặc nhiều regions**
- Sử dụng **Administrator Account** triển khai vào **Target Accounts**
- Tự động deploy vào accounts/OUs mới khi được thêm vào Organization

**Khi nào dùng StackSet:**
- Baseline security config (CloudTrail, Config, GuardDuty) trên tất cả accounts
- IAM roles chuẩn (cross-account access roles)
- Compliance guardrails phải có trên mọi account
- VPC baseline networking trên nhiều regions

**Hai chế độ deployment:**
1. **Self-managed StackSets** — Phải tạo IAM roles thủ công ở cả admin và target accounts
2. **Service-managed StackSets** — Dùng AWS Organizations, tự động hóa hoàn toàn (khuyên dùng)

```bash
# Deploy StackSet vào toàn bộ OU
aws cloudformation create-stack-set \
  --stack-set-name "baseline-security" \
  --template-url https://s3.amazonaws.com/templates/security-baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name "baseline-security" \
  --deployment-targets OrganizationalUnitIds=["ou-xxxx-yyyyyyyy"] \
  --regions us-east-1 eu-west-1 ap-southeast-1
```

---

### Q20. Change Set trong CloudFormation là gì? Khi nào bắt buộc phải dùng?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

**Change Set** (Bộ Thay Đổi) là tính năng cho phép xem trước (preview) những thay đổi sẽ xảy ra với Stack trước khi thực sự áp dụng — tương tự `terraform plan`.

**Workflow:**
```
1. Tạo Change Set từ template mới
2. Review: thêm gì, xóa gì, thay đổi gì
3. Chú ý: "Replacement" (phá hủy và tạo lại) vs "Modify" (sửa tại chỗ)
4. Approve → Execute Change Set
```

**Khi bắt buộc phải dùng:**
- **Replacement actions** — tài nguyên sẽ bị xóa và tạo lại (RDS, EC2 có thể gây downtime)
- **Production updates** — mọi thay đổi production phải qua Change Set
- **Shared resources** — VPC, Security Groups đang được nhiều stack dùng

**Cách đọc Change Set output:**

```
Action   LogicalResourceId    ResourceType          Replacement
──────   ─────────────────    ─────────────         ───────────
Modify   WebServerGroup       AWS::AutoScaling::... False      ← An toàn
Modify   LaunchConfig         AWS::AutoScaling::... True       ← NGUY HIỂM - xóa/tạo lại
Add      NewBucket            AWS::S3::Bucket       N/A        ← Thêm mới
Remove   OldFunction          AWS::Lambda::...      N/A        ← XÓA
```

**Điểm Cộng:** Đề cập `Replacement: True` thường do thay đổi **immutable properties** (thuộc tính không thể sửa tại chỗ) như `DBInstanceClass` của RDS hay `ImageId` của EC2 — những thứ này phải tạo resource mới.

---

### Q21. CloudFormation Drift Detection là gì? Xử lý thế nào?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

**Drift** (Trôi Dạt) xảy ra khi cấu hình thực tế của tài nguyên **khác** với cấu hình được định nghĩa trong CloudFormation template — thường do ai đó thay đổi thủ công qua Console hoặc CLI.

**Drift Detection** phát hiện và báo cáo sự khác biệt này.

**Trigger Drift Detection:**
```bash
aws cloudformation detect-stack-drift \
  --stack-name my-production-stack

# Xem kết quả
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id-từ-lệnh-trên>

# Xem chi tiết resource nào bị drift
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-production-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

**Xử lý khi phát hiện drift:**

**Option 1 — Import vào Stack** (Nếu thay đổi thủ công là hợp lệ):
- Cập nhật template để phản ánh cấu hình thực tế
- Dùng `drift:import-stacks-to-stack-set` hoặc update stack

**Option 2 — Revert thủ công** (Nếu thay đổi trái phép):
- Xác định ai thay đổi qua CloudTrail
- Revert tài nguyên về đúng config trong template
- Tạo SCP/IAM policy ngăn thay đổi trực tiếp

**Option 3 — CloudFormation tự remediate:**
- Update stack với `--use-previous-template` để force tài nguyên về trạng thái template

**Prevention:**
- IAM policy deny direct resource modification: deny `ec2:ModifySecurityGroup` trừ CloudFormation role
- AWS Config Rule: phát hiện thay đổi ngoài CloudFormation

---

### Q22. CDK vs CloudFormation — khi nào nên chuyển sang CDK?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

**CDK — AWS Cloud Development Kit** (Bộ Công Cụ Phát Triển Đám Mây AWS) cho phép định nghĩa infrastructure bằng ngôn ngữ lập trình thực sự (Python, TypeScript, Java, Go...), compile ra CloudFormation template.

**Nên chuyển sang CDK khi:**

1. **Cần abstraction và reuse** — Viết Construct một lần, dùng nhiều chỗ. Ví dụ: `StandardMicroservice` construct = Lambda + API Gateway + CloudWatch Alarms + IAM Role chuẩn.

2. **Logic phức tạp** — CloudFormation YAML không có loop, condition thực. CDK có thể: `for service in services: create_alarms(service)`

3. **Unit testing** — CDK có `assertions` library cho phép viết test kiểm tra infrastructure trước khi deploy.

4. **Team developer là chính** — Developer biết Python/TypeScript hơn YAML.

**Giữ CloudFormation khi:**
- Template đơn giản, stable, ít thay đổi
- Team operations không biết lập trình
- Cần integration với external systems đọc CloudFormation natively
- StackSets phức tạp (CDK hỗ trợ nhưng CDK Pipelines cần setup thêm)

**CDK Unit Test ví dụ:**
```python
from aws_cdk.assertions import Template

def test_s3_bucket_has_versioning():
    stack = MyStack(app, "TestStack")
    template = Template.from_stack(stack)
    template.has_resource_properties("AWS::S3::Bucket", {
        "VersioningConfiguration": {"Status": "Enabled"}
    })
```

---

## PHẦN 6: ORGANIZATIONS & MULTI-ACCOUNT

---

### Q23. Tại sao nên dùng nhiều AWS account thay vì một account lớn?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**5 lý do chính:**

**1. Blast Radius** (Bán Kính Nổ) — Khi sự cố xảy ra (bị hack, developer nhầm xóa), chỉ ảnh hưởng account đó. Production an toàn khỏi lỗi trong dev.

**2. Security Boundary** (Ranh Giới Bảo Mật) — IAM permissions không thể cross account mà không có explicit trust. Developer dev không thể vô tình touch production.

**3. Cost Attribution** (Phân Bổ Chi Phí) — Mỗi account = một billing unit rõ ràng. Biết chính xác team A tốn bao nhiêu không cần tag phức tạp.

**4. Service Limits** (Giới Hạn Dịch Vụ) — Mỗi account có quota riêng. EC2 limit của dev không ăn vào quota của prod.

**5. Compliance Isolation** (Cách Ly Tuân Thủ) — PCI-DSS scope chỉ cần cover account chứa cardholder data, không phải toàn bộ tổ chức.

**Điểm Cộng:** Trích dẫn AWS Well-Architected Framework — multi-account là best practice chính thức của AWS cho production workloads.

---

### Q24. SCP — Service Control Policy là gì? Allow list vs Deny list?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅✅

**Đáp Án:**

**SCP — Service Control Policy** (Chính Sách Kiểm Soát Dịch Vụ) là chính sách trong AWS Organizations giới hạn **quyền tối đa** (maximum permissions) mà IAM identities trong một account/OU có thể có.

**Quan trọng:** SCP không *cấp* quyền — chỉ *giới hạn* quyền. Phải có cả SCP cho phép VÀ IAM policy cấp quyền.

**Allow List** (Danh Sách Cho Phép):
- Xóa FullAWSAccess mặc định
- Chỉ allow những service/action được chỉ định rõ
- Mặc định: từ chối tất cả, phải cho phép từng thứ
- Phù hợp: môi trường có kiểm soát chặt, sandbox

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*", "ec2:*", "rds:*"],
    "Resource": "*"
  }]
}
```

**Deny List** (Danh Sách Từ Chối):
- Giữ FullAWSAccess mặc định
- Thêm explicit Deny cho những gì không được phép
- Mặc định: cho phép tất cả, deny những ngoại lệ
- Phù hợp: hầu hết trường hợp, ít ảnh hưởng đến teams

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": [
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "config:DeleteConfigRule",
      "organizations:LeaveOrganization"
    ],
    "Resource": "*"
  }]
}
```

**SCP Inheritance (Kế Thừa):**
```
Root SCP: Deny LeaveOrganization
├── OU: Production
│   └── SCP: Deny Delete* + Deny DisableCloudTrail
│       └── Account: Prod-A → Có cả 2 deny từ Root + OU
└── OU: Development
    └── Account: Dev-B → Chỉ có deny từ Root
```

---

### Q25. Consolidated Billing trong Organizations hoạt động thế nào?

**Mức độ:** ⭐⭐ Trung Bình | **Hay hỏi:** ✅✅

**Đáp Án:**

**Consolidated Billing** (Hóa Đơn Hợp Nhất) gộp chi phí từ tất cả member accounts về một hóa đơn duy nhất thanh toán bởi Management Account.

**Lợi ích chính:**

**1. Volume Discounts** (Giảm Giá Theo Khối Lượng) — S3, EC2 data transfer, CloudFront tính phí theo tier. Dùng cộng dồn:
- Dev account dùng 1TB S3 = $23/TB
- Prod account dùng 4TB S3 = $22/TB (tier tiếp theo)
- Cộng dồn: 5TB → toàn bộ hưởng mức $21.5/TB

**2. Reserved Instance & Savings Plans Sharing** (Chia Sẻ Reserved Instance) — RI mua ở account A tự động apply cho instances cùng loại ở account B trong Organization.

**3. Single Payer** (Một Người Thanh Toán) — Một thẻ tín dụng, một hóa đơn, đơn giản hóa kế toán.

**Điểm Cộng:** Đề cập **Cost Allocation Tags** (Thẻ Phân Bổ Chi Phí) — tag tài nguyên với `Project`, `Team`, `Environment` để phân tích chi phí theo dimension dù billing hợp nhất.

---

### Q26. Thiết kế OU structure cho doanh nghiệp 50 teams. Approach của bạn là gì?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

**Nguyên tắc thiết kế:**
1. Tổ chức theo **policy requirements** (yêu cầu chính sách), không phải theo org chart
2. Accounts trong cùng OU nên có **cùng SCPs**
3. Tránh nested OUs quá sâu (≤ 5 cấp)

**Template OU Structure chuẩn:**

```
Root
├── OU: Security        → SCP: deny disable audit services
│   ├── Log Archive     → S3 centralized logs, ai cũng push vào đây
│   └── Audit           → Security tooling: Security Hub master, GuardDuty master
│
├── OU: Infrastructure  → SCP: deny create default VPC
│   ├── Shared Services → Transit Gateway, DNS, Directory Services
│   └── Network         → VPCs, peering, PrivateLink endpoints
│
├── OU: Workloads       → SCP: require tags, deny expensive reserved instances
│   ├── OU: Production  → SCP: deny Delete* without approval tag
│   │   ├── Team-A-Prod
│   │   ├── Team-B-Prod
│   │   └── ...
│   └── OU: SDLC        → SCP: auto-shutdown, budget cap
│       ├── Team-A-Dev
│       ├── Team-A-Staging
│       └── ...
│
├── OU: Exceptions      → Accounts cần policy đặc biệt (legacy, ISV)
│   └── Legacy-App      → Giữ nguyên, không apply standard SCPs
│
└── OU: Sandbox         → SCP: budget hard limit, deny production-grade, 30-day TTL
    └── Developer sandboxes
```

**Điểm Cộng:** Nhắc đến **Account Vending Machine** (Máy Bán Tài Khoản) pattern — Account Factory tự động tạo account mới theo template chuẩn khi team mới request.

---

## PHẦN 7: CONTROL TOWER & LANDING ZONE

---

### Q27. Control Tower Landing Zone gồm những thành phần gì?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

Khi thiết lập Control Tower, Landing Zone (Vùng Hạ Cánh) tự động tạo:

**Accounts được tạo tự động:**
1. **Management Account** — Root của Organization (account bạn đang dùng để setup)
2. **Log Archive Account** — Nhận tất cả CloudTrail và Config logs, S3 bucket centralized
3. **Audit Account** — Security tooling, cross-account read-only access, SNS notifications

**Services được bật tự động:**
- **AWS Organizations** với OU structure chuẩn
- **CloudTrail Organization Trail** → Log Archive Account
- **AWS Config** với Organization Aggregator → Audit Account
- **IAM Identity Center (SSO)** — Single Sign-On cho tất cả accounts
- **GuardDuty** (tùy chọn)
- **Security Hub** (tùy chọn)

**Guardrails được áp dụng:**
- **Mandatory Guardrails** (Bắt Buộc, không thể tắt): Không cho disable CloudTrail, không cho thay đổi Log Archive S3 bucket, không cho leave Organization
- **Strongly Recommended** (Khuyến Nghị Mạnh): Deny public S3, deny unencrypted EBS
- **Elective** (Tùy Chọn): Nhiều guardrails khác có thể bật/tắt

---

### Q28. Preventive Guardrails vs Detective Guardrails trong Control Tower khác nhau thế nào?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

**Preventive Guardrails** (Rào Chắn Phòng Ngừa):
- Dùng **SCPs** để **ngăn chặn** hành động vi phạm xảy ra ngay từ đầu
- Nếu cố gắng thực hiện action bị cấm → API call bị từ chối ngay lập tức
- Ví dụ: Deny disable CloudTrail, Deny create IAM users (phải dùng IAM roles)

**Detective Guardrails** (Rào Chắn Phát Hiện):
- Dùng **AWS Config Rules** để **phát hiện** vi phạm sau khi đã xảy ra
- Không ngăn chặn hành động, chỉ báo cáo compliance status
- Dashboard Control Tower hiển thị accounts nào đang vi phạm
- Ví dụ: Detect S3 bucket không có versioning, detect EC2 không có required tags

**Khi nào dùng loại nào:**
- Preventive: Dùng cho policy critical — không được vi phạm trong bất kỳ trường hợp nào (disable audit, leave org)
- Detective: Dùng cho policy quan trọng nhưng có thể có exception hợp lý (tag, encryption)

**Triển khai:**
```
Preventive → SCP attached to OU → Block via Organizations API
Detective  → Config Rule → Evaluate → Report to Control Tower Dashboard
           → Optionally → SSM Automation → Auto-remediate
```

---

## PHẦN 8: COST GOVERNANCE & FINOPS

---

### Q29. Thiết kế cost governance strategy cho môi trường đa account. Approach của bạn?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅✅

**Đáp Án:**

**FinOps** (Financial Operations — Vận Hành Tài Chính Đám Mây) cho đa account cần 3 layer:

**Layer 1: Visibility** (Khả Năng Nhìn Thấy)

- **AWS Cost Explorer** với Consolidated Billing view
- **Cost Allocation Tags** (Thẻ Phân Bổ Chi Phí): `Project`, `Team`, `Environment`, `CostCenter` → Tag Policy trong Organizations enforce
- **AWS Cost and Usage Report (CUR)** → S3 → Athena → QuickSight dashboard

**Layer 2: Control** (Kiểm Soát)

- **AWS Budgets** mỗi account: alert khi 80% và 100% budget
- **Budget Actions**: khi 100% budget → deny IAM policy (không cho tạo thêm resources)
- **SCP** trong Sandbox OU: deny `m5.4xlarge`, deny `db.r5.2xlarge` (chặn instances đắt tiền)

**Layer 3: Optimization** (Tối Ưu)

- **Cost Anomaly Detection** — ML detect spike bất thường
- **Compute Optimizer** — Rightsizing recommendations
- **Savings Plans** cho baseline workload (EC2 + Fargate)
- **Trusted Advisor** — Idle resource identification

**Governance Process:**
```
Monthly FinOps Review:
1. Review Cost Explorer: ai tăng nhiều nhất?
2. Check Anomaly Detection alerts: bất thường nào chưa giải quyết?
3. Review Trusted Advisor: có idle resources nào mới?
4. Optimize: mua thêm Savings Plans nếu commitment coverage < 80%
```

---

### Q30. Cost Anomaly Detection hoạt động thế nào? Khác gì với Budget Alerts?

**Mức độ:** ⭐⭐⭐ Nâng Cao | **Hay hỏi:** ✅

**Đáp Án:**

**AWS Budget Alerts** (Cảnh Báo Ngân Sách):
- So sánh chi tiêu thực tế với **ngưỡng cố định** do bạn đặt
- Ví dụ: Alert khi chi tiêu > $1,000/tháng
- Biết khi nào vượt budget, nhưng không biết **tại sao** và **có bất thường không**

**Cost Anomaly Detection** (Phát Hiện Chi Phí Bất Thường):
- Dùng **machine learning** (học máy) học baseline chi tiêu theo service, account, tag
- Phát hiện khi chi tiêu **khác bất thường** so với pattern lịch sử — dù không vượt budget
- Có thể phát hiện: EC2 instance bị để quên, Lambda bị loop, data transfer đột biến
- Alert theo **ngưỡng $ hoặc %** deviation từ baseline

**Ví dụ thực tế:**
```
Budget Alert: Tháng 1 budget = $5,000
  → Chi tiêu ngày 25 = $4,200 → Alert "80% budget"
  → Nhưng không biết ngày 20 có spike bất thường $800 một ngày không

Anomaly Detection:
  → Ngày 20: Lambda cost = $800 (baseline: $50/ngày) → Alert ngay lập tức!
  → "Anomaly detected: Lambda exceeds expected spend by $750 (1500%)"
  → Hành động ngay trước khi tháng kết thúc
```

**Cấu hình Monitor:**
```bash
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ServiceMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DevTeamAlerts",
    "MonitorArnList": ["arn:aws:ce::123456789:anomalymonitor/xxx"],
    "Subscribers": [{"Address": "devteam@company.com", "Type": "EMAIL"}],
    "Threshold": 50,
    "Frequency": "IMMEDIATE"
  }'
```

**Điểm Cộng:** Đề cập sử dụng **cả hai** — Budget Alerts như hard limit, Anomaly Detection như early warning system. Bổ sung nhau hoàn hảo.

---

## Tóm Tắt — Key Points Cho Mỗi Chủ Đề

### CloudWatch
- Metrics = số, Logs = text; Metric Filter chuyển log pattern thành metric
- Composite Alarm tránh alert spam
- Container Insights cho ECS/EKS; Agent cần để có Memory metric

### CloudTrail
- 3 event types: Management (free), Data (paid), Insights (paid)
- Organization Trail = best practice, member không tắt được
- CloudTrail = "ai làm gì"; Config = "cấu hình có đúng không"

### AWS Config
- Managed Rules (200+) vs Custom Rules (Lambda/Guard)
- Conformance Pack = bundle rules theo framework (CIS/PCI/HIPAA)
- Auto-Remediation qua SSM Automation

### SSM
- Session Manager > bastion host: không cần port 22, có audit log
- Parameter Store (free, config) vs Secrets Manager (paid, auto-rotation)
- Run Command = one-shot; Automation = multi-step workflow

### CloudFormation
- Stack = 1 account/region; StackSet = multi-account/region
- Change Set = preview trước khi apply (như terraform plan)
- Drift = tài nguyên bị thay đổi ngoài IaC

### Organizations
- Multi-account = blast radius, security boundary, cost clarity
- SCP = giới hạn tối đa, không cấp quyền
- Deny List (giữ FullAccess + deny ngoại lệ) phổ biến hơn Allow List

### Control Tower
- Landing Zone tự tạo: Log Archive, Audit accounts + Organization Trail
- Preventive (SCP) = ngăn chặn; Detective (Config Rule) = phát hiện

### Cost Governance
- Budget = ngưỡng cố định; Anomaly Detection = ML phát hiện bất thường
- Tag Policy enforce → Cost Explorer filter theo team/project
- Savings Plans cho baseline; Spot cho flexible workloads

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
