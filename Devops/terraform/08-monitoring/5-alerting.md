# Alerting — Cảnh Báo Khi Có Thay Đổi Ngoài Dự Kiến

> Alerting — Cảnh Báo — trong ngữ cảnh Terraform và hạ tầng là hệ thống tự động thông báo khi hạ tầng có biến động bất thường: drift phát hiện, chi phí vượt ngưỡng, thay đổi cấu hình nhạy cảm, hoặc apply ngoài giờ. Mục tiêu là giảm MTTD — Mean Time To Detect — Thời Gian Phát Hiện Trung Bình.

---

## 🎯 Những Gì Cần Cảnh Báo

### Ma Trận Alert Priority — Mức Độ Ưu Tiên Cảnh Báo

| Sự Kiện | Mức Độ | Kênh | Thời Gian Phản Hồi |
|---------|--------|------|---------------------|
| Drift trong production security group | P0 — Critical | PagerDuty + Slack | < 15 phút |
| Apply production ngoài giờ hành chính | P0 — Critical | PagerDuty + Slack | < 15 phút |
| State file bị xóa | P0 — Critical | PagerDuty + Slack | < 15 phút |
| Chi phí tăng > 50% trong ngày | P1 — High | Slack + Email | < 1 giờ |
| Drift trong staging | P1 — High | Slack | < 4 giờ |
| Resource không có required tags | P2 — Medium | Slack | < 24 giờ |
| Terraform plan fail trong CI/CD | P2 — Medium | Slack | < 4 giờ |
| Chi phí dự báo vượt budget 80% | P3 — Low | Email | < 48 giờ |

---

## 🔔 Kênh Cảnh Báo

### 1. Slack Notifications

**Webhook đơn giản:**

```bash
# Gửi thông báo cơ bản
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  --data '{
    "text": "⚠️ Drift detected in production infrastructure!"
  }'
```

**Rich message với Block Kit:**

```bash
# Gửi thông báo có cấu trúc đẹp
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  --data '{
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": "⚠️ Infrastructure Drift Detected — Phát Hiện Lệch Cấu Hình"
        }
      },
      {
        "type": "section",
        "fields": [
          {
            "type": "mrkdwn",
            "text": "*Environment:*\nProduction"
          },
          {
            "type": "mrkdwn", 
            "text": "*Detected At:*\n2026-05-12 10:30 UTC"
          },
          {
            "type": "mrkdwn",
            "text": "*Resources Affected:*\naws_security_group.web-sg"
          },
          {
            "type": "mrkdwn",
            "text": "*Severity:*\n🔴 P0 — Critical"
          }
        ]
      },
      {
        "type": "actions",
        "elements": [
          {
            "type": "button",
            "text": {"type": "plain_text", "text": "View Plan Output"},
            "url": "https://github.com/company/infra/actions/runs/123456"
          }
        ]
      }
    ]
  }'
```

### 2. PagerDuty Integration

```bash
# Tạo PagerDuty incident cho P0 alerts
send_pagerduty_alert() {
  local SUMMARY="$1"
  local DETAILS="$2"
  local SEVERITY="${3:-critical}"  # critical, error, warning, info
  
  curl -X POST "https://events.pagerduty.com/v2/enqueue" \
    -H "Content-Type: application/json" \
    --data "{
      \"routing_key\": \"$PAGERDUTY_INTEGRATION_KEY\",
      \"event_action\": \"trigger\",
      \"payload\": {
        \"summary\": \"$SUMMARY\",
        \"severity\": \"$SEVERITY\",
        \"source\": \"terraform-monitoring\",
        \"custom_details\": $DETAILS
      }
    }"
}

# Sử dụng
send_pagerduty_alert \
  "Production drift detected in security group" \
  '{"environment": "production", "resource": "aws_security_group.web"}' \
  "critical"
```

### 3. Email Notifications Qua AWS SNS

```hcl
# sns.tf — Tạo SNS topics cho từng severity
resource "aws_sns_topic" "infrastructure_alerts_critical" {
  name = "infra-alerts-critical"
  
  tags = {
    Environment = var.environment
    Purpose     = "infrastructure-alerting"
  }
}

resource "aws_sns_topic" "infrastructure_alerts_warning" {
  name = "infra-alerts-warning"
}

# Subscribe email vào SNS topic
resource "aws_sns_topic_subscription" "email_critical" {
  topic_arn = aws_sns_topic.infrastructure_alerts_critical.arn
  protocol  = "email"
  endpoint  = "oncall-team@company.com"
}

# Có thể subscribe Lambda để forward sang Slack/PagerDuty
resource "aws_sns_topic_subscription" "lambda_critical" {
  topic_arn = aws_sns_topic.infrastructure_alerts_critical.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.alert_forwarder.arn
}
```

---

## 🤖 Alerting Trong CI/CD

### GitHub Actions — Alert Tích Hợp Hoàn Chỉnh

```yaml
# .github/workflows/terraform-monitoring.yml
name: Terraform Monitoring & Alerting

on:
  schedule:
    - cron: '0 */6 * * *'  # Mỗi 6 giờ
  workflow_dispatch:

jobs:
  drift-check-and-alert:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-southeast-1

      - name: Terraform Init
        run: terraform init
        working-directory: ./infrastructure/prod

      - name: Run Drift Check
        id: drift
        working-directory: ./infrastructure/prod
        run: |
          set +e
          terraform plan \
            -detailed-exitcode \
            -no-color \
            -out=drift.tfplan \
            2>&1 | tee plan-output.txt
          EXIT_CODE=$?
          echo "exit_code=$EXIT_CODE" >> $GITHUB_OUTPUT
          
          # Parse changed resources
          CHANGES=$(grep -E "^  [#~+-]" plan-output.txt | head -20 || echo "")
          echo "changes<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
          
          exit 0  # Không fail job
      
      # Alert nếu có drift
      - name: Alert on Drift Detected
        if: steps.drift.outputs.exit_code == '2'
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_INFRA_WEBHOOK }}
        run: |
          CHANGES_ESCAPED=$(echo '${{ steps.drift.outputs.changes }}' | \
            python3 -c "import sys, json; print(json.dumps(sys.stdin.read()))")
          
          curl -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data "{
              \"blocks\": [
                {
                  \"type\": \"header\",
                  \"text\": {
                    \"type\": \"plain_text\",
                    \"text\": \"🚨 DRIFT DETECTED — Production Infrastructure\"
                  }
                },
                {
                  \"type\": \"section\",
                  \"text\": {
                    \"type\": \"mrkdwn\",
                    \"text\": \"*Changes detected:*\n\`\`\`${CHANGES_ESCAPED}\`\`\`\"
                  }
                },
                {
                  \"type\": \"actions\",
                  \"elements\": [{
                    \"type\": \"button\",
                    \"text\": {\"type\": \"plain_text\", \"text\": \"View Details\"},
                    \"url\": \"${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"
                  }]
                }
              ]
            }"

      # Alert nếu plan lỗi (có thể là provider issue, credential issue...)
      - name: Alert on Plan Error
        if: steps.drift.outputs.exit_code == '1'
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_INFRA_WEBHOOK }}
        run: |
          curl -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data '{
              "text": "❌ *Terraform plan FAILED* — Production monitoring check cannot run. Immediate investigation needed!",
              "attachments": [{
                "color": "danger",
                "fields": [{
                  "title": "Run",
                  "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                }]
              }]
            }'
```

### Alert Khi Apply Xảy Ra Ngoài Giờ (Off-Hours)

```yaml
# .github/workflows/production-apply-guard.yml
name: Production Apply Guard — Bảo Vệ Apply Production

on:
  workflow_run:
    workflows: ["Terraform Apply Production"]
    types:
      - completed

jobs:
  check-apply-timing:
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion == 'success'
    
    steps:
      - name: Check if apply was off-hours
        run: |
          # Giờ hành chính: 9AM-6PM UTC+7 (2AM-11AM UTC), Thứ 2 - Thứ 6
          HOUR=$(date -u +%H)
          DOW=$(date -u +%u)  # 1=Monday, 7=Sunday
          
          IS_OFF_HOURS=false
          
          # Cuối tuần
          if [ "$DOW" -ge 6 ]; then
            IS_OFF_HOURS=true
          fi
          
          # Ngoài giờ hành chính (UTC)
          if [ "$HOUR" -lt 2 ] || [ "$HOUR" -ge 11 ]; then
            IS_OFF_HOURS=true
          fi
          
          echo "is_off_hours=$IS_OFF_HOURS" >> $GITHUB_ENV

      - name: Alert Off-Hours Apply
        if: env.is_off_hours == 'true'
        run: |
          curl -X POST "${{ secrets.SLACK_SECURITY_WEBHOOK }}" \
            -H 'Content-type: application/json' \
            --data '{
              "text": "🌙 *Off-Hours Production Apply Detected* — Áp dụng sản xuất ngoài giờ!\n\nActor: ${{ github.event.workflow_run.actor.login }}\nTime: '"$(date -u)"'\nRun: ${{ github.event.workflow_run.html_url }}\n\n@oncall Please verify this was intentional."
            }'
```

---

## 📊 AWS CloudWatch Alerts

### Alert Cho CloudTrail Events

```hcl
# cloudwatch-alerts.tf

# Tạo metric filter cho unauthorized API calls
resource "aws_cloudwatch_log_metric_filter" "unauthorized_api_calls" {
  name           = "UnauthorizedApiCalls"
  pattern        = "{ ($.errorCode = \"*UnauthorizedAccess*\") || ($.errorCode = \"AccessDenied\") }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "UnauthorizedApiCallCount"
    namespace = "TerraformAudit"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "unauthorized_api_calls" {
  alarm_name          = "unauthorized-api-calls"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "UnauthorizedApiCallCount"
  namespace           = "TerraformAudit"
  period              = "300"  # 5 phút
  statistic           = "Sum"
  threshold           = "5"    # Alert nếu > 5 lần trong 5 phút
  alarm_description   = "Phát hiện nhiều unauthorized API calls — có thể là credential leak hoặc misconfiguration"
  treat_missing_data  = "notBreaching"

  alarm_actions = [aws_sns_topic.infrastructure_alerts_critical.arn]
}

# Alert khi security group thay đổi
resource "aws_cloudwatch_log_metric_filter" "security_group_changes" {
  name    = "SecurityGroupChanges"
  pattern = "{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "SecurityGroupChangeCount"
    namespace = "TerraformAudit"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "security_group_changes" {
  alarm_name          = "security-group-changes"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "SecurityGroupChangeCount"
  namespace           = "TerraformAudit"
  period              = "300"
  statistic           = "Sum"
  threshold           = "0"  # Alert với BẤT KỲ thay đổi nào
  alarm_description   = "Phát hiện thay đổi Security Group — review để xác nhận hợp lệ"

  alarm_actions = [aws_sns_topic.infrastructure_alerts_warning.arn]
}
```

### Cost Anomaly Detection — Phát Hiện Chi Phí Bất Thường

```hcl
# cost-alerts.tf

# AWS Cost Anomaly Detection
resource "aws_ce_anomaly_monitor" "infrastructure" {
  name              = "terraform-infrastructure-monitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "daily_alert" {
  name      = "daily-cost-anomaly-alert"
  frequency = "DAILY"
  
  monitor_arn_list = [aws_ce_anomaly_monitor.infrastructure.arn]
  
  subscriber {
    address = aws_sns_topic.infrastructure_alerts_warning.arn
    type    = "SNS"
  }

  threshold_expression {
    and {
      dimension {
        key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
        values        = ["100"]  # Alert nếu anomaly > $100
        match_options = ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }
}

# Budget alert với SNS
resource "aws_budgets_budget" "monthly_infrastructure" {
  name         = "monthly-infrastructure-budget"
  budget_type  = "COST"
  limit_amount = "50000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.infrastructure_alerts_warning.arn]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_sns_topic_arns  = [aws_sns_topic.infrastructure_alerts_critical.arn]
  }
}
```

---

## 🔧 Lambda Alert Handler — Xử Lý Và Định Tuyến Cảnh Báo

```python
# lambda/alert_handler.py
"""
Lambda function nhận events từ nhiều nguồn (CloudWatch, Cost Anomaly...)
và định tuyến đến đúng kênh (Slack, PagerDuty) dựa trên severity.
"""
import json
import os
import boto3
import urllib.request
from enum import Enum


class Severity(Enum):
    CRITICAL = "critical"    # PagerDuty + Slack
    HIGH     = "high"        # Slack + Email
    MEDIUM   = "medium"      # Slack
    LOW      = "low"         # Email


SLACK_WEBHOOKS = {
    Severity.CRITICAL: os.environ['SLACK_WEBHOOK_CRITICAL'],
    Severity.HIGH:     os.environ['SLACK_WEBHOOK_HIGH'],
    Severity.MEDIUM:   os.environ['SLACK_WEBHOOK_MEDIUM'],
    Severity.LOW:      os.environ.get('SLACK_WEBHOOK_LOW', ''),
}

EMOJI_MAP = {
    Severity.CRITICAL: "🚨",
    Severity.HIGH:     "⚠️",
    Severity.MEDIUM:   "⚡",
    Severity.LOW:      "ℹ️",
}


def send_slack(webhook_url: str, message: dict) -> None:
    data = json.dumps(message).encode('utf-8')
    req = urllib.request.Request(
        webhook_url,
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)


def trigger_pagerduty(summary: str, details: dict) -> None:
    payload = {
        "routing_key": os.environ['PAGERDUTY_KEY'],
        "event_action": "trigger",
        "payload": {
            "summary": summary,
            "severity": "critical",
            "source": "terraform-monitoring",
            "custom_details": details
        }
    }
    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(
        "https://events.pagerduty.com/v2/enqueue",
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)


def classify_alert(event: dict) -> tuple[Severity, str, dict]:
    """Phân loại alert và trả về (severity, summary, details)"""
    
    source = event.get('source', '')
    detail = event.get('detail', {})
    
    # CloudWatch Alarm
    if source == 'aws.cloudwatch':
        alarm_name = detail.get('alarmName', '')
        state = detail.get('state', {}).get('value', '')
        
        if 'security-group' in alarm_name and state == 'ALARM':
            return (
                Severity.HIGH,
                f"Security Group thay đổi phát hiện",
                {"alarm": alarm_name, "state": state}
            )
        
        if 'unauthorized-api' in alarm_name and state == 'ALARM':
            return (
                Severity.CRITICAL,
                "Unauthorized API calls detected",
                {"alarm": alarm_name}
            )
    
    # Cost Anomaly
    if source == 'aws.ce':
        impact = detail.get('anomalyDetails', {}).get('impact', {})
        total_impact = float(impact.get('totalImpact', 0))
        
        if total_impact > 1000:
            return (
                Severity.CRITICAL,
                f"Cost anomaly: +${total_impact:.0f}",
                {"impact": total_impact, "details": detail}
            )
        elif total_impact > 100:
            return (
                Severity.HIGH,
                f"Cost anomaly: +${total_impact:.0f}",
                {"impact": total_impact}
            )
    
    # Default
    return (Severity.MEDIUM, "Infrastructure alert", event)


def lambda_handler(event, context):
    # Nếu đến từ SNS, unwrap
    if 'Records' in event:
        for record in event['Records']:
            if record.get('EventSource') == 'aws:sns':
                message = json.loads(record['Sns']['Message'])
                process_alert(message)
    else:
        process_alert(event)
    
    return {'statusCode': 200}


def process_alert(event: dict) -> None:
    severity, summary, details = classify_alert(event)
    emoji = EMOJI_MAP[severity]
    
    # Tạo Slack message
    slack_message = {
        "text": f"{emoji} *{summary}*",
        "attachments": [{
            "color": "danger" if severity in (Severity.CRITICAL, Severity.HIGH) else "warning",
            "fields": [
                {"title": k, "value": str(v), "short": True}
                for k, v in details.items()
                if isinstance(v, (str, int, float))
            ]
        }]
    }
    
    # Gửi Slack
    webhook = SLACK_WEBHOOKS.get(severity)
    if webhook:
        send_slack(webhook, slack_message)
    
    # PagerDuty cho Critical
    if severity == Severity.CRITICAL:
        trigger_pagerduty(summary, details)
```

---

## 📋 Alert Runbook — Hướng Dẫn Xử Lý Khi Nhận Alert

### Runbook Cho Drift Alert

```markdown
## Runbook: Production Drift Alert

### Khi Nhận Alert
1. Acknowledge alert trong Slack/PagerDuty (< 5 phút)
2. Click link "View Plan Output" trong alert
3. Đọc plan output để hiểu thay đổi là gì

### Triage
- Thay đổi security group (ingress/egress rules)?
  → P0: Escalate ngay, có thể là security incident
  
- Thay đổi instance size/type?
  → P1: Có thể ai đó resize manual để xử lý tải
  
- Thay đổi tags?
  → P2: Ít nguy hiểm, schedule fix trong giờ hành chính

### Xử Lý
Option A - Re-apply Terraform (nếu thay đổi không được phép):
  1. Tạo Jira ticket ghi lại incident
  2. terraform apply -auto-approve
  3. Verify không có regression
  4. Document root cause

Option B - Update Code (nếu thay đổi hợp lý):
  1. Tạo PR với thay đổi tương ứng trong Terraform code
  2. Review và merge
  3. Verify plan = no changes

### Post-Incident
- Document trong post-mortem nếu P0/P1
- Xem xét thêm guardrails để ngăn tái diễn
```

---

## 🧪 Testing Alerts — Kiểm Tra Cảnh Báo

```bash
# Test Slack webhook hoạt động
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  --data '{"text": "✅ Alert system test — Hệ thống cảnh báo hoạt động bình thường"}'

# Test CloudWatch alarm bằng cách force ALARM state
aws cloudwatch set-alarm-state \
  --alarm-name "security-group-changes" \
  --state-value ALARM \
  --state-reason "Manual test"

# Verify alert nhận được, sau đó reset
aws cloudwatch set-alarm-state \
  --alarm-name "security-group-changes" \
  --state-value OK \
  --state-reason "Test complete"
```

---

## 📊 Alert Fatigue — Tránh Cảnh Báo Quá Nhiều

> Alert fatigue — Mệt mỏi do cảnh báo — xảy ra khi có quá nhiều alerts dẫn đến team bỏ qua hoặc bịt tắt thông báo. Đây là anti-pattern nguy hiểm.

### Nguyên Tắc Chống Alert Fatigue

```
1. Mỗi alert phải actionable (có hành động cụ thể để xử lý)
   ❌ Alert: "Terraform ran successfully" — Không cần biết
   ✅ Alert: "Drift detected in security group" — Cần xử lý

2. Đúng người nhận đúng alert
   ❌ Mọi người nhận mọi alert
   ✅ Security team nhận security alerts
      Dev team nhận code/deploy alerts
      Finance team nhận cost alerts

3. Suppress — Tắt tạm — alerts trong maintenance windows
   # Tắt alerts trong khoảng thời gian maintenance
   aws cloudwatch disable-alarm-actions --alarm-names "drift-detection"
   # ... maintenance ...
   aws cloudwatch enable-alarm-actions --alarm-names "drift-detection"

4. Theo dõi alert noise — Tiếng ồn cảnh báo
   Nếu alert bị close mà không có action > 70% → Cần review lại threshold
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Bạn xây dựng alerting cho Terraform như thế nào? Làm sao tránh alert fatigue?**

> Tôi phân loại alerts theo hai chiều: severity (critical/high/medium/low) và actionability (có cần hành động ngay không). Chỉ route critical alerts đến PagerDuty để wake on-call. Medium/low alerts vào Slack với context đủ để xử lý không cần escalate. Tôi theo dõi tỷ lệ "alert → no action" — nếu vượt 30% thì review threshold. Quan trọng nhất: mỗi alert phải có runbook gắn kèm để người nhận biết phải làm gì.

**Q: Apply Terraform xảy ra lúc 3 giờ sáng — bạn biết không? Làm sao?**

> Với CI/CD pipeline đúng cách, production apply chỉ xảy ra qua GitHub Actions trigger bởi approved PR. Tôi có CloudTrail capturing terraform-ci-role API calls và CloudWatch alarm cho off-hours activity. GitHub Actions workflow cũng gửi Slack notification cho mọi production apply kèm actor và timestamp. Nếu apply xảy ra ngoài giờ và không qua CI/CD, PagerDuty alert sẽ wake on-call ngay lập tức.

---

## 🔗 Tài Liệu Tham Khảo

- [AWS CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [AWS Cost Anomaly Detection](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)
- [PagerDuty Events API v2](https://developer.pagerduty.com/api-reference/YXBpOjI3NDgyNjU-pager-duty-v2-events-api)
- [Slack Block Kit Builder](https://app.slack.com/block-kit-builder)

---

**Phần Này Đã Hoàn Thành:** [08-monitoring/README.md](./README.md)

**Phần Tiếp Theo:** [09-troubleshooting/](../09-troubleshooting/) — Xử lý sự cố hạ tầng

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
