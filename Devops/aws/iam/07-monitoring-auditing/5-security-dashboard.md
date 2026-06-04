# Security Dashboard — CloudWatch Dashboards Và Alarms Cho Bảo Mật

> Xây dựng **Security Dashboard** (Bảng Điều Khiển Bảo Mật) trực quan giúp team phát hiện bất thường ngay lập tức và duy trì tầm nhìn bảo mật 24/7.

---

## 🎯 Tại Sao Cần Security Dashboard?

Một Security Dashboard tốt cho phép bạn:

- **Nhận biết ngay** khi bảo mật bị vi phạm — trong phút, không phải giờ
- **Xu hướng dài hạn** — phát hiện leo thang quyền dần dần qua nhiều tuần
- **Chứng minh tuân thủ** — báo cáo cho auditors bằng dữ liệu trực quan
- **Giảm alert fatigue** (mệt mỏi cảnh báo) — ưu tiên những gì thực sự quan trọng

---

## 🏗️ Kiến Trúc Security Monitoring Stack

```
CloudTrail Logs ──┐
VPC Flow Logs ────┤
                  ▼
          CloudWatch Logs
                  │
    ┌─────────────┼──────────────┐
    ▼             ▼              ▼
Metric         Logs           Alarms
Filters       Insights         │
    │             │            ▼
    ▼             │     SNS Topics
CloudWatch        │         │
Metrics           │    ┌────┴───────────┐
    │             │    ▼    ▼           ▼
    └─────────────┤  Email Slack  PagerDuty
                  │
                  ▼
          CloudWatch Dashboard
          (Security Operations)
```

---

## 📊 CloudWatch Metric Filters — Bộ Lọc Metric

**Metric Filters** (Bộ Lọc Metric) chuyển đổi log text thành CloudWatch metrics có thể tạo alarms.

### Cách Hoạt Động

```
CloudTrail Log Entry (text JSON)
           │
           ▼ Metric Filter áp dụng pattern
           │
           ▼ Match? → tăng counter metric lên 1
           │
           ▼
  CloudWatch Metric (số)
           │
           ▼ Alarm khi metric > threshold
```

### Bộ Metric Filters Bảo Mật Cần Thiết

#### 1. Root Account Usage — Sử Dụng Root Account

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootAccountUsageCount,\
    metricNamespace=CISBenchmark,\
    metricValue=1,\
    defaultValue=0
```

#### 2. CloudTrail Configuration Changes — Thay Đổi Cấu Hình CloudTrail

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "CloudTrailChanges" \
  --filter-pattern '{ ($.eventName = CreateTrail) || ($.eventName = UpdateTrail) || ($.eventName = DeleteTrail) || ($.eventName = StartLogging) || ($.eventName = StopLogging) }' \
  --metric-transformations \
    metricName=CloudTrailChangesCount,\
    metricNamespace=CISBenchmark,\
    metricValue=1,\
    defaultValue=0
```

#### 3. Console Login Failures — Đăng Nhập Thất Bại

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "ConsoleLoginFailures" \
  --filter-pattern '{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }' \
  --metric-transformations \
    metricName=ConsoleLoginFailureCount,\
    metricNamespace=SecurityMetrics,\
    metricValue=1,\
    defaultValue=0
```

#### 4. IAM Policy Changes — Thay Đổi IAM Policy

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "IAMPolicyChanges" \
  --filter-pattern '{($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=SetDefaultPolicyVersion)||($.eventName=AttachRolePolicy)||($.eventName=DetachRolePolicy)||($.eventName=AttachUserPolicy)||($.eventName=DetachUserPolicy)||($.eventName=AttachGroupPolicy)||($.eventName=DetachGroupPolicy)}' \
  --metric-transformations \
    metricName=IAMPolicyChangesCount,\
    metricNamespace=CISBenchmark,\
    metricValue=1,\
    defaultValue=0
```

#### 5. Network Gateway Changes — Thay Đổi Network Gateway

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "NetworkGatewayChanges" \
  --filter-pattern '{ ($.eventName = CreateCustomerGateway) || ($.eventName = DeleteCustomerGateway) || ($.eventName = AttachInternetGateway) || ($.eventName = CreateInternetGateway) || ($.eventName = DeleteInternetGateway) || ($.eventName = DetachInternetGateway) }' \
  --metric-transformations \
    metricName=NetworkGatewayChangesCount,\
    metricNamespace=CISBenchmark,\
    metricValue=1,\
    defaultValue=0
```

#### 6. Security Group Changes — Thay Đổi Security Group

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "SecurityGroupChanges" \
  --filter-pattern '{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup)}' \
  --metric-transformations \
    metricName=SecurityGroupChangesCount,\
    metricNamespace=CISBenchmark,\
    metricValue=1,\
    defaultValue=0
```

#### 7. Unauthorized API Calls — API Calls Không Được Phép

```bash
aws logs put-metric-filter \
  --log-group-name "CloudTrail/SecurityEvents" \
  --filter-name "UnauthorizedAPICalls" \
  --filter-pattern '{ ($.errorCode = "*UnauthorizedAccess") || ($.errorCode = "AccessDenied*") }' \
  --metric-transformations \
    metricName=UnauthorizedAPICallsCount,\
    metricNamespace=SecurityMetrics,\
    metricValue=1,\
    defaultValue=0
```

---

## 🚨 CloudWatch Alarms — Cảnh Báo

### Tạo Alarms Cho Security Metrics

```bash
# Alarm: Root account được dùng (ngay lập tức, không có ngưỡng delay)
aws cloudwatch put-metric-alarm \
  --alarm-name "CIS-1.1-RootAccountUsage" \
  --alarm-description "[CIS 1.1] Root account usage detected" \
  --metric-name RootAccountUsageCount \
  --namespace CISBenchmark \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:P1-SecurityAlerts \
  --ok-actions arn:aws:sns:ap-southeast-1:123456789012:SecurityOK

# Alarm: Nhiều login failure (brute force detection)
aws cloudwatch put-metric-alarm \
  --alarm-name "SEC-ConsoleLoginBruteForce" \
  --alarm-description "5+ console login failures trong 5 phút — nghi ngờ brute force" \
  --metric-name ConsoleLoginFailureCount \
  --namespace SecurityMetrics \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:P2-SecurityAlerts

# Alarm: Unauthorized API calls spike
aws cloudwatch put-metric-alarm \
  --alarm-name "SEC-UnauthorizedAPISpike" \
  --alarm-description "Spike trong unauthorized API calls — có thể đang probe permissions" \
  --metric-name UnauthorizedAPICallsCount \
  --namespace SecurityMetrics \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:P2-SecurityAlerts
```

### Bảng Cảnh Báo Theo Mức Độ Ưu Tiên

| Alarm | Threshold | Priority | SNS Topic |
|---|---|---|---|
| Root account usage | ≥ 1 trong 5 phút | 🔴 P1 | `P1-SecurityAlerts` |
| CloudTrail stopped | ≥ 1 | 🔴 P1 | `P1-SecurityAlerts` |
| KMS key deletion | ≥ 1 | 🔴 P1 | `P1-SecurityAlerts` |
| Console login failure | ≥ 5 trong 5 phút | 🟡 P2 | `P2-SecurityAlerts` |
| IAM policy changes | ≥ 1 | 🟡 P2 | `P2-SecurityAlerts` |
| Security Group changes | ≥ 3 trong 5 phút | 🟡 P2 | `P2-SecurityAlerts` |
| Unauthorized API calls | ≥ 10 trong 5 phút | 🟡 P2 | `P2-SecurityAlerts` |
| Network gateway changes | ≥ 1 | 🟠 P3 | `P3-SecurityAlerts` |
| VPC changes | ≥ 1 | 🟠 P3 | `P3-SecurityAlerts` |

---

## 📈 CloudWatch Dashboard — Bảng Điều Khiển

### Tạo Security Operations Dashboard

```python
import boto3
import json

cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

dashboard_body = {
    "widgets": [
        # Row 1: Tiêu đề
        {
            "type": "text",
            "x": 0, "y": 0, "width": 24, "height": 1,
            "properties": {
                "markdown": "# 🔐 AWS Security Operations Dashboard\n**Cập nhật tự động mỗi 5 phút** | Tầm nhìn bảo mật realtime"
            }
        },

        # Row 2: Số liệu tổng quan (Single value widgets)
        {
            "type": "metric",
            "x": 0, "y": 1, "width": 4, "height": 4,
            "properties": {
                "title": "Root Account Usage (24h)",
                "view": "singleValue",
                "metrics": [["CISBenchmark", "RootAccountUsageCount"]],
                "period": 86400,
                "stat": "Sum",
                "setPeriodToTimeRange": True,
                "sparkline": True,
                "trend": True
            }
        },
        {
            "type": "metric",
            "x": 4, "y": 1, "width": 4, "height": 4,
            "properties": {
                "title": "Login Failures (24h)",
                "view": "singleValue",
                "metrics": [["SecurityMetrics", "ConsoleLoginFailureCount"]],
                "period": 86400,
                "stat": "Sum",
                "sparkline": True
            }
        },
        {
            "type": "metric",
            "x": 8, "y": 1, "width": 4, "height": 4,
            "properties": {
                "title": "IAM Changes (24h)",
                "view": "singleValue",
                "metrics": [["CISBenchmark", "IAMPolicyChangesCount"]],
                "period": 86400,
                "stat": "Sum",
                "sparkline": True
            }
        },
        {
            "type": "metric",
            "x": 12, "y": 1, "width": 4, "height": 4,
            "properties": {
                "title": "Unauthorized Calls (1h)",
                "view": "singleValue",
                "metrics": [["SecurityMetrics", "UnauthorizedAPICallsCount"]],
                "period": 3600,
                "stat": "Sum",
                "sparkline": True
            }
        },
        {
            "type": "alarm",
            "x": 16, "y": 1, "width": 8, "height": 4,
            "properties": {
                "title": "Active Security Alarms",
                "alarms": [
                    "arn:aws:cloudwatch:ap-southeast-1:123456789012:alarm:CIS-1.1-RootAccountUsage",
                    "arn:aws:cloudwatch:ap-southeast-1:123456789012:alarm:SEC-ConsoleLoginBruteForce",
                    "arn:aws:cloudwatch:ap-southeast-1:123456789012:alarm:SEC-UnauthorizedAPISpike"
                ]
            }
        },

        # Row 3: Time series charts
        {
            "type": "metric",
            "x": 0, "y": 5, "width": 12, "height": 6,
            "properties": {
                "title": "Console Login Activity (Last 24h)",
                "view": "timeSeries",
                "metrics": [
                    ["SecurityMetrics", "ConsoleLoginFailureCount", {"label": "Failures", "color": "#d62728"}],
                    ["SecurityMetrics", "ConsoleLoginSuccessCount", {"label": "Success", "color": "#2ca02c"}]
                ],
                "period": 300,
                "stat": "Sum",
                "yAxis": {"left": {"min": 0}}
            }
        },
        {
            "type": "metric",
            "x": 12, "y": 5, "width": 12, "height": 6,
            "properties": {
                "title": "API Error Rate (Last 24h)",
                "view": "timeSeries",
                "metrics": [
                    ["SecurityMetrics", "UnauthorizedAPICallsCount", {"label": "Unauthorized", "color": "#d62728"}],
                    ["CISBenchmark", "IAMPolicyChangesCount", {"label": "IAM Changes", "color": "#ff7f0e"}],
                    ["CISBenchmark", "SecurityGroupChangesCount", {"label": "SG Changes", "color": "#1f77b4"}]
                ],
                "period": 300,
                "stat": "Sum"
            }
        },

        # Row 4: Logs Insights widget
        {
            "type": "log",
            "x": 0, "y": 11, "width": 24, "height": 6,
            "properties": {
                "title": "Recent Security Events (Top 20)",
                "query": "SOURCE 'CloudTrail/SecurityEvents' | fields @timestamp, userIdentity.type, userIdentity.userName, eventName, sourceIPAddress, errorCode | filter errorCode like 'AccessDenied' or userIdentity.type = 'Root' | sort @timestamp desc | limit 20",
                "region": "ap-southeast-1",
                "view": "table"
            }
        }
    ]
}

response = cloudwatch.put_dashboard(
    DashboardName='SecurityOperations',
    DashboardBody=json.dumps(dashboard_body)
)
```

---

## 🏢 Terraform — Toàn Bộ Security Monitoring Stack

```hcl
# SNS Topics cho các mức độ cảnh báo
resource "aws_sns_topic" "p1_alerts" {
  name = "P1-SecurityAlerts"
  kms_master_key_id = aws_kms_key.sns.id
}

resource "aws_sns_topic" "p2_alerts" {
  name = "P2-SecurityAlerts"
}

# Subscriptions
resource "aws_sns_topic_subscription" "p1_pagerduty" {
  topic_arn = aws_sns_topic.p1_alerts.arn
  protocol  = "https"
  endpoint  = var.pagerduty_endpoint
}

resource "aws_sns_topic_subscription" "p1_email" {
  topic_arn = aws_sns_topic.p1_alerts.arn
  protocol  = "email"
  endpoint  = "security-team@company.com"
}

# CloudWatch Log Group cho CloudTrail
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "CloudTrail/SecurityEvents"
  retention_in_days = 365
  kms_key_id        = aws_kms_key.cloudwatch.arn
}

# Metric Filters
locals {
  security_metric_filters = {
    RootAccountUsage = {
      pattern    = "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"
      metric     = "RootAccountUsageCount"
      namespace  = "CISBenchmark"
    }
    CloudTrailChanges = {
      pattern    = "{ ($.eventName = CreateTrail) || ($.eventName = UpdateTrail) || ($.eventName = DeleteTrail) || ($.eventName = StopLogging) }"
      metric     = "CloudTrailChangesCount"
      namespace  = "CISBenchmark"
    }
    IAMPolicyChanges = {
      pattern    = "{ ($.eventName = PutUserPolicy) || ($.eventName = PutGroupPolicy) || ($.eventName = PutRolePolicy) || ($.eventName = AttachRolePolicy) || ($.eventName = DetachRolePolicy) || ($.eventName = CreatePolicy) || ($.eventName = DeletePolicy) }"
      metric     = "IAMPolicyChangesCount"
      namespace  = "CISBenchmark"
    }
    SecurityGroupChanges = {
      pattern    = "{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }"
      metric     = "SecurityGroupChangesCount"
      namespace  = "CISBenchmark"
    }
  }
}

resource "aws_cloudwatch_log_metric_filter" "security" {
  for_each       = local.security_metric_filters
  name           = each.key
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = each.value.pattern

  metric_transformation {
    name          = each.value.metric
    namespace     = each.value.namespace
    value         = "1"
    default_value = "0"
  }
}

# Alarms
resource "aws_cloudwatch_metric_alarm" "root_account_usage" {
  alarm_name          = "CIS-1.1-RootAccountUsage"
  alarm_description   = "[CIS 3.3] Root account usage detected"
  metric_name         = "RootAccountUsageCount"
  namespace           = "CISBenchmark"
  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.p1_alerts.arn]
  ok_actions          = [aws_sns_topic.p2_alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "cloudtrail_changes" {
  alarm_name          = "CIS-CloudTrailChanges"
  alarm_description   = "[CIS 3.5] CloudTrail configuration changes detected"
  metric_name         = "CloudTrailChangesCount"
  namespace           = "CISBenchmark"
  statistic           = "Sum"
  period              = 300
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  treat_missing_data  = "notBreaching"
  alarm_actions       = [aws_sns_topic.p1_alerts.arn]
}
```

---

## 📐 CIS Benchmark Metric Filters Mapping

**CIS AWS Foundations Benchmark** (Chuẩn Nền Tảng AWS của CIS — Center for Internet Security) yêu cầu các metric filters và alarms cụ thể:

| CIS Control | Metric Filter | Mô tả |
|---|---|---|
| CIS 3.1 | UnauthorizedAPICalls | Unauthorized API calls |
| CIS 3.2 | ConsoleSignInWithoutMFA | Login không có MFA |
| CIS 3.3 | RootAccountUsage | Root account dùng |
| CIS 3.4 | IAMPolicyChanges | IAM policy thay đổi |
| CIS 3.5 | CloudTrailChanges | CloudTrail config thay đổi |
| CIS 3.6 | ConsoleAuthFailures | Login failure |
| CIS 3.7 | DisableOrDeleteCMK | KMS key bị xóa/tắt |
| CIS 3.8 | S3BucketPolicyChanges | S3 bucket policy thay đổi |
| CIS 3.9 | AWSConfigChanges | Config thay đổi |
| CIS 3.10 | SecurityGroupChanges | Security Group thay đổi |
| CIS 3.11 | NACLChanges | NACL thay đổi |
| CIS 3.12 | NetworkGatewayChanges | Network gateway thay đổi |
| CIS 3.13 | RouteTableChanges | Route table thay đổi |
| CIS 3.14 | VPCChanges | VPC thay đổi |

---

## 📱 Tích Hợp Thông Báo

### Gửi Alert Vào Slack

```python
# Lambda function: security-alert-to-slack
import json
import urllib.request

SLACK_WEBHOOK = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

def lambda_handler(event, context):
    sns_message = json.loads(event['Records'][0]['Sns']['Message'])
    
    alarm_name = sns_message.get('AlarmName', 'Unknown')
    alarm_desc = sns_message.get('AlarmDescription', '')
    new_state = sns_message.get('NewStateValue', '')
    reason = sns_message.get('NewStateReason', '')
    account_id = sns_message.get('AWSAccountId', '')
    region = sns_message.get('Region', '')
    
    color = {
        'ALARM': 'danger',
        'OK': 'good',
        'INSUFFICIENT_DATA': 'warning'
    }.get(new_state, 'warning')
    
    emoji = {
        'ALARM': '🚨',
        'OK': '✅',
        'INSUFFICIENT_DATA': '⚠️'
    }.get(new_state, '⚠️')
    
    payload = {
        "username": "AWS Security Monitor",
        "icon_emoji": ":shield:",
        "attachments": [{
            "color": color,
            "title": f"{emoji} {alarm_name}",
            "text": alarm_desc,
            "fields": [
                {"title": "State", "value": new_state, "short": True},
                {"title": "Account", "value": account_id, "short": True},
                {"title": "Region", "value": region, "short": True},
                {"title": "Reason", "value": reason, "short": False}
            ],
            "footer": "AWS CloudWatch Alarm",
            "ts": int(__import__('time').time())
        }]
    }
    
    req = urllib.request.Request(
        SLACK_WEBHOOK,
        data=json.dumps(payload).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)
    
    return {"statusCode": 200}
```

---

## 📊 Security Metrics Nên Theo Dõi Hàng Ngày

| Metric | Target | Cảnh báo khi |
|---|---|---|
| Root login attempts | 0 | > 0 |
| Failed console logins | < 5/ngày | > 20/ngày |
| IAM policy changes | < 10/ngày | > 50/ngày |
| Security Group changes | < 20/ngày | > 100/ngày |
| Unauthorized API calls | < 50/ngày | > 200/ngày |
| CloudTrail disruptions | 0 | > 0 |
| Access Analyzer findings | 0 | > 0 trong 24h không resolve |
| Config NON_COMPLIANT resources | < 5% | > 10% |

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Thiết kế SIEM (Security Information and Event Management — Quản Lý Thông Tin Và Sự Kiện Bảo Mật) trên AWS mà không dùng third-party tools?**

> Kiến trúc native AWS SIEM: CloudTrail + VPC Flow Logs → CloudWatch Logs → Metric Filters cho detection. CloudTrail Lake hoặc Athena cho historical analysis và correlation. EventBridge cho event-driven automation. Security Hub để aggregate và prioritize findings từ GuardDuty, Inspector, Macie, Config, Access Analyzer. CloudWatch Dashboard cho real-time visualization. SNS + Lambda cho notification và auto-remediation. Chi phí thấp hơn Splunk/Elastic SIEM nhưng phù hợp cho 90% use cases.

**Q: Làm thế nào giảm alert fatigue trong security monitoring?**

> Ba chiến lược: (1) **Tiering** — phân loại P1/P2/P3, P1 wake on-call, P2 business hours, P3 weekly digest. (2) **Suppression rules** — tự động archive findings đã review như CloudFront access patterns. (3) **Contextual enrichment** — Lambda enrich alerts với thông tin thêm (ví dụ: user này có thường xuyên làm việc ngoài giờ không?) trước khi notify. Kết hợp với machine learning baseline từ CloudTrail Insights để phân biệt bất thường thực sự vs noise bình thường.

**Q: Nên giữ CloudWatch Logs bao lâu?**

> Theo CIS Benchmark và hầu hết compliance frameworks: tối thiểu 90 ngày online (hot storage), 1 năm total. Thực tế tốt nhất: 90 ngày trong CloudWatch Logs ($0.03/GB/month), export sang S3 + Glacier cho 7 năm ($0.004/GB/month). Dùng CloudWatch Logs subscription filter để export real-time sang S3 thông qua Kinesis Firehose, không cần manual export.

---

**Kết Thúc Module:** [README.md](README.md) — Quay lại tổng quan module

---

**Cập Nhật Lần Cuối:** 2026-05-16
