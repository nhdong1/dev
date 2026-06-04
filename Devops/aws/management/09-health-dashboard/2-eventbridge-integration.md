# EventBridge Integration — Tự Động Hóa Phản Hồi Health Events

> **EventBridge** (Cầu Nối Sự Kiện) là xương sống của mọi chiến lược tự động hóa với AWS Health. Thay vì chờ kỹ sư phát hiện sự cố trong Console, bạn xây dựng pipeline tự động nhận event → xử lý → phản hồi, giảm MTTR (Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình) từ hàng giờ xuống vài phút.

---

## 📚 Mục Lục

1. [Tại Sao EventBridge?](#tại-sao-eventbridge)
2. [Kiến Trúc Cơ Bản](#kiến-trúc-cơ-bản)
3. [Event Pattern — Lọc Sự Kiện](#event-pattern--lọc-sự-kiện)
4. [Pattern 1: Notification (Thông Báo)](#pattern-1-notification-thông-báo)
5. [Pattern 2: Auto-Remediation (Tự Động Khắc Phục)](#pattern-2-auto-remediation-tự-động-khắc-phục)
6. [Pattern 3: Ticket Creation (Tạo Ticket Tự Động)](#pattern-3-ticket-creation-tạo-ticket-tự-động)
7. [Pattern 4: Multi-Account Centralized Handling](#pattern-4-multi-account-centralized-handling)
8. [Step Functions cho Complex Workflows](#step-functions-cho-complex-workflows)
9. [Monitoring & Observability của Pipeline](#monitoring--observability-của-pipeline)
10. [Infrastructure as Code với Terraform](#infrastructure-as-code-với-terraform)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao EventBridge?

### Vấn Đề Với Approach Thủ Công

```
Không có EventBridge:
─────────────────────
1. Kỹ sư phải kiểm tra Console định kỳ (hoặc nhận email AWS)
2. Phát hiện muộn — đôi khi sau khi instance đã bị retire
3. Response inconsistent — mỗi người làm khác nhau
4. Không có audit trail rõ ràng về "ai đã làm gì khi nào"
5. Khó scale khi fleet lớn hơn
```

### Với EventBridge Integration

```
Có EventBridge:
──────────────
1. Health event → EventBridge → Lambda (tức thì, không cần người)
2. Response trong vài giây đến vài phút
3. Consistent — cùng code, cùng quy trình mỗi lần
4. Mọi hành động được log trong CloudWatch Logs
5. Scale tự động bất kể fleet có bao nhiêu instance
```

---

## Kiến Trúc Cơ Bản

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS Health Service                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Event: AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED               │ │
│  │ Affected: [i-0abc123, i-0def456]                           │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────┘
                               │ EventBridge publishes
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Amazon EventBridge (Default Bus)                │
│                                                                  │
│  Rule: source=aws.health, detail-type=AWS Health Event           │
│  Filter: eventTypeCode=AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED     │
└──────────────────┬──────────────────────────────────────────────┘
                   │ Route to targets
         ┌─────────┼──────────────────────────┐
         │         │                          │
         ▼         ▼                          ▼
    ┌─────────┐ ┌──────────┐         ┌───────────────┐
    │   SNS   │ │  Lambda  │         │ Step Functions│
    │  Topic  │ │ Function │         │  State Machine│
    └────┬────┘ └────┬─────┘         └───────┬───────┘
         │           │                       │
    ┌────▼───┐  ┌────▼──────────┐   ┌────────▼──────────────────┐
    │ Slack  │  │ JIRA Ticket   │   │ 1. Snapshot EBS           │
    │ Email  │  │ ServiceNow    │   │ 2. Stop instance          │
    │ PagerDuty│ │ PagerDuty    │   │ 3. Start instance (migrate)│
    └────────┘  └───────────────┘   │ 4. Verify new hardware    │
                                    │ 5. Update DNS/ELB         │
                                    └───────────────────────────┘
```

---

## Event Pattern — Lọc Sự Kiện

### Pattern Cơ Bản — Bắt Mọi Health Events

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"]
}
```

### Pattern Nâng Cao — Lọc Theo Service và Category

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"],
  "detail": {
    "service": ["EC2", "RDS", "LAMBDA", "EKS"],
    "eventTypeCategory": ["issue", "scheduledChange"]
  }
}
```

### Pattern Cụ Thể — Chỉ EC2 Retirement

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"],
  "detail": {
    "eventTypeCode": [
      "AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED",
      "AWS_EC2_INSTANCE_REBOOT_SCHEDULED"
    ]
  }
}
```

### Pattern Production-Grade — Nhiều Event Types

```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"],
  "detail": {
    "eventTypeCode": [
      "AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED",
      "AWS_EC2_INSTANCE_REBOOT_SCHEDULED",
      "AWS_RDS_MAINTENANCE_SCHEDULED",
      "AWS_LAMBDA_RUNTIME_DEPRECATION_SCHEDULED",
      "AWS_RDS_SSL_CA_CERTIFICATE_EXPIRY",
      "AWS_VPC_OPERATIONAL_ISSUE",
      "AWS_EC2_OPERATIONAL_ISSUE"
    ]
  }
}
```

---

## Pattern 1: Notification (Thông Báo)

Đây là pattern **đơn giản nhất** và nên là bước đầu tiên với mọi tổ chức.

### Kiến Trúc

```
Health Event → EventBridge Rule → SNS Topic → Lambda → Slack/PagerDuty/Email
```

### CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS Health Events Notification Pipeline

Resources:
  # SNS Topic nhận Health events
  HealthAlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: aws-health-alerts
      Subscription:
        - Protocol: email
          Endpoint: ops-team@company.com

  # EventBridge Rule bắt Health events
  HealthEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: aws-health-all-events
      Description: Capture all AWS Health events
      EventPattern:
        source:
          - aws.health
        detail-type:
          - AWS Health Event
        detail:
          eventTypeCategory:
            - issue
            - scheduledChange
      State: ENABLED
      Targets:
        - Id: NotifyOpsTeam
          Arn: !Ref HealthAlertTopic
          InputTransformer:
            InputPathsMap:
              eventType: "$.detail.eventTypeCode"
              service: "$.detail.service"
              region: "$.region"
              account: "$.account"
              startTime: "$.detail.startTime"
            InputTemplate: |
              "⚠️ AWS Health Alert
              Service: <service>
              Region: <region>
              Account: <account>
              Event: <eventType>
              Time: <startTime>
              Please check AWS Health Dashboard for details."

  # Permission cho EventBridge publish tới SNS
  SnsTopicPolicy:
    Type: AWS::SNS::TopicPolicy
    Properties:
      Topics:
        - !Ref HealthAlertTopic
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sns:Publish
            Resource: !Ref HealthAlertTopic
```

### Lambda Formatter cho Slack

```python
import json
import urllib.request
import os

SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']
SLACK_CHANNEL = os.environ.get('SLACK_CHANNEL', '#aws-health-alerts')

SEVERITY_EMOJI = {
    'issue': '🔴',
    'scheduledChange': '🟡',
    'accountNotification': '🔵'
}

def format_slack_message(event):
    detail = event.get('detail', {})
    category = detail.get('eventTypeCategory', 'unknown')
    service = detail.get('service', 'Unknown')
    event_type = detail.get('eventTypeCode', 'Unknown')
    region = event.get('region', 'Unknown')
    account = event.get('account', 'Unknown')

    affected = detail.get('affectedEntities', [])
    affected_summary = ', '.join(
        e.get('entityValue', '') for e in affected[:3]
    )
    if len(affected) > 3:
        affected_summary += f' +{len(affected) - 3} more'

    emoji = SEVERITY_EMOJI.get(category, '⚠️')

    return {
        "channel": SLACK_CHANNEL,
        "attachments": [
            {
                "color": "danger" if category == "issue" else "warning",
                "title": f"{emoji} AWS Health Alert: {service}",
                "fields": [
                    {"title": "Event Type", "value": event_type, "short": True},
                    {"title": "Region", "value": region, "short": True},
                    {"title": "Account", "value": account, "short": True},
                    {"title": "Category", "value": category, "short": True},
                    {"title": "Affected Resources", "value": affected_summary or "N/A", "short": False}
                ],
                "footer": "AWS Health Dashboard",
                "actions": [
                    {
                        "type": "button",
                        "text": "View in Console",
                        "url": f"https://{region}.console.aws.amazon.com/health/home"
                    }
                ]
            }
        ]
    }

def handler(event, context):
    message = format_slack_message(event)
    payload = json.dumps(message).encode('utf-8')

    req = urllib.request.Request(
        SLACK_WEBHOOK_URL,
        data=payload,
        headers={'Content-Type': 'application/json'}
    )

    with urllib.request.urlopen(req) as response:
        return {'statusCode': response.status}
```

---

## Pattern 2: Auto-Remediation (Tự Động Khắc Phục)

Pattern này **tự động thực hiện hành động** khi nhận Health event, không cần con người can thiệp.

### Use Case: Tự Động Migrate EC2 Instance Trước Retirement

```
Health Event (EC2 Retirement)
    → EventBridge Rule
    → Lambda (kiểm tra environment tag)
        ├── Nếu tag Environment=dev: stop/start ngay lập tức
        ├── Nếu tag Environment=staging: lên lịch trong 24 giờ
        └── Nếu tag Environment=production: gửi PagerDuty + chờ approval
```

### Lambda Auto-Remediation Code

```python
import boto3
import json
import os
from datetime import datetime, timezone

ec2 = boto3.client('ec2')
ssm = boto3.client('ssm')
sns = boto3.client('sns')

ALERT_TOPIC_ARN = os.environ['ALERT_TOPIC_ARN']
AUTO_MIGRATE_ENVS = ['dev', 'test', 'staging']  # Tự động migrate những env này

def get_instance_environment(instance_id):
    """Lấy tag Environment của instance."""
    response = ec2.describe_instances(InstanceIds=[instance_id])
    tags = response['Reservations'][0]['Instances'][0].get('Tags', [])
    for tag in tags:
        if tag['Key'] == 'Environment':
            return tag['Value'].lower()
    return 'unknown'

def migrate_instance(instance_id):
    """Stop và Start instance để migrate sang phần cứng mới."""
    print(f"Migrating instance {instance_id}...")

    # Stop instance
    ec2.stop_instances(InstanceIds=[instance_id])
    waiter = ec2.get_waiter('instance_stopped')
    waiter.wait(InstanceIds=[instance_id])
    print(f"  Instance {instance_id} stopped.")

    # Start instance (trigger hardware migration)
    ec2.start_instances(InstanceIds=[instance_id])
    waiter = ec2.get_waiter('instance_running')
    waiter.wait(InstanceIds=[instance_id])
    print(f"  Instance {instance_id} started on new hardware.")

    # Get new public IP
    response = ec2.describe_instances(InstanceIds=[instance_id])
    new_ip = response['Reservations'][0]['Instances'][0].get('PublicIpAddress', 'N/A (private only)')
    print(f"  New IP: {new_ip}")

    return new_ip

def send_alert(subject, message):
    """Gửi alert qua SNS."""
    sns.publish(
        TopicArn=ALERT_TOPIC_ARN,
        Subject=subject,
        Message=message
    )

def handler(event, context):
    detail = event.get('detail', {})

    # Chỉ xử lý retirement events
    if detail.get('eventTypeCode') != 'AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED':
        return {'status': 'skipped', 'reason': 'not a retirement event'}

    affected_entities = detail.get('affectedEntities', [])
    results = []

    for entity in affected_entities:
        instance_id = entity['entityValue']
        environment = get_instance_environment(instance_id)

        print(f"Processing {instance_id} (env={environment})")

        if environment in AUTO_MIGRATE_ENVS:
            # Tự động migrate
            try:
                new_ip = migrate_instance(instance_id)
                results.append({
                    'instance': instance_id,
                    'action': 'auto-migrated',
                    'new_ip': new_ip,
                    'environment': environment
                })
                send_alert(
                    f"✅ Auto-migrated {instance_id}",
                    f"Instance {instance_id} ({environment}) đã được tự động migrate.\nNew IP: {new_ip}"
                )
            except Exception as e:
                results.append({
                    'instance': instance_id,
                    'action': 'migration-failed',
                    'error': str(e)
                })
                send_alert(
                    f"❌ Migration failed: {instance_id}",
                    f"Không thể tự động migrate {instance_id}.\nLỗi: {str(e)}\nCần can thiệp thủ công."
                )
        else:
            # Production: chỉ thông báo, không tự động
            results.append({
                'instance': instance_id,
                'action': 'alert-only',
                'environment': environment
            })
            send_alert(
                f"⚠️ Manual action required: {instance_id}",
                f"Instance {instance_id} ({environment}) sẽ bị retire.\n"
                f"Cần lên lịch maintenance window để migrate thủ công.\n"
                f"Thao tác: Stop → Start instance (KHÔNG dùng Reboot)."
            )

    print(f"Processed {len(results)} instances: {json.dumps(results)}")
    return {'status': 'completed', 'results': results}
```

---

## Pattern 3: Ticket Creation (Tạo Ticket Tự Động)

Tạo ticket trong hệ thống ITSM (IT Service Management — Quản Lý Dịch Vụ CNTT) như JIRA hoặc ServiceNow khi nhận Health event.

### Kiến Trúc

```
Health Event → EventBridge → Lambda → JIRA API → Ticket được tạo tự động
                                    → Tag EC2 instance với Ticket ID
                                    → Assign cho team on-call
```

### Lambda JIRA Integration

```python
import boto3
import json
import os
import urllib.request
import urllib.parse
import base64

JIRA_BASE_URL = os.environ['JIRA_BASE_URL']
JIRA_USER = os.environ['JIRA_USER']
JIRA_API_TOKEN_PARAM = os.environ['JIRA_API_TOKEN_PARAM']
JIRA_PROJECT_KEY = os.environ['JIRA_PROJECT_KEY']

ssm = boto3.client('ssm')
ec2 = boto3.client('ec2')

def get_jira_token():
    response = ssm.get_parameter(
        Name=JIRA_API_TOKEN_PARAM,
        WithDecryption=True
    )
    return response['Parameter']['Value']

def create_jira_ticket(summary, description, priority='High'):
    token = get_jira_token()
    credentials = base64.b64encode(f"{JIRA_USER}:{token}".encode()).decode()

    payload = {
        "fields": {
            "project": {"key": JIRA_PROJECT_KEY},
            "summary": summary,
            "description": description,
            "issuetype": {"name": "Incident"},
            "priority": {"name": priority},
            "labels": ["aws-health", "automated"]
        }
    }

    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(
        f"{JIRA_BASE_URL}/rest/api/2/issue",
        data=data,
        headers={
            'Content-Type': 'application/json',
            'Authorization': f'Basic {credentials}'
        }
    )

    with urllib.request.urlopen(req) as response:
        result = json.loads(response.read())
        return result['key']  # Ví dụ: OPS-1234

def tag_instances_with_ticket(instance_ids, ticket_id):
    if instance_ids:
        ec2.create_tags(
            Resources=instance_ids,
            Tags=[{'Key': 'HealthTicket', 'Value': ticket_id}]
        )

def handler(event, context):
    detail = event['detail']
    event_type = detail['eventTypeCode']
    service = detail['service']
    region = event['region']

    affected = detail.get('affectedEntities', [])
    instance_ids = [e['entityValue'] for e in affected]

    priority_map = {
        'issue': 'Critical',
        'scheduledChange': 'High',
        'accountNotification': 'Medium'
    }
    priority = priority_map.get(detail.get('eventTypeCategory', ''), 'Medium')

    summary = f"[AWS Health] {service}: {event_type} @ {region}"
    description = (
        f"**AWS Health Event Detected**\n\n"
        f"- Service: {service}\n"
        f"- Event Type: {event_type}\n"
        f"- Region: {region}\n"
        f"- Account: {event['account']}\n"
        f"- Affected Resources: {', '.join(instance_ids) or 'N/A'}\n\n"
        f"**Action Required:** Check AWS Health Dashboard for details and recommended actions."
    )

    ticket_id = create_jira_ticket(summary, description, priority)
    print(f"Created JIRA ticket: {ticket_id}")

    # Gắn ticket ID vào instance tags để dễ tracking
    if instance_ids and service == 'EC2':
        tag_instances_with_ticket(instance_ids, ticket_id)

    return {'ticket_id': ticket_id, 'affected_resources': len(instance_ids)}
```

---

## Pattern 4: Multi-Account Centralized Handling

### Kiến Trúc Tập Trung

Trong môi trường multi-account, tập trung xử lý Health events tại một Security/Operations account.

```
Dev Account ──────────────────────────────────────┐
  Health Event → EventBridge → EventBus (local)   │
                                    │              │
                                    └──────────────┤
Staging Account ───────────────────────────────── ┤
  Health Event → EventBridge → EventBus (local)   │  Cross-account
                                    │              │  event routing
                                    └──────────────┤
Production Account ────────────────────────────── ┘
  Health Event → EventBridge → EventBus (local)   │
                                    │              │
                                    └──────────────▼
                              Security Account: Central EventBus
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                         Lambda (processing)      SQS Queue
                              │                  (buffering)
                    ┌─────────┤
                    │         │         │
                 JIRA      Slack    PagerDuty
                 Ticket    Alert    Incident
```

### Thiết Lập Cross-Account Event Bus

**Bước 1: Tại Security Account — Tạo Event Bus nhận từ member accounts**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::111111111111:root",
          "arn:aws:iam::222222222222:root",
          "arn:aws:iam::333333333333:root"
        ]
      },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:SECURITY_ACCOUNT_ID:event-bus/health-central"
    }
  ]
}
```

**Bước 2: Tại mỗi Member Account — Forward health events sang central bus**

```yaml
# CloudFormation tại member account
HealthForwardRule:
  Type: AWS::Events::Rule
  Properties:
    Name: forward-health-to-central
    EventPattern:
      source:
        - aws.health
      detail-type:
        - AWS Health Event
    State: ENABLED
    Targets:
      - Id: CentralBus
        Arn: arn:aws:events:us-east-1:SECURITY_ACCOUNT_ID:event-bus/health-central
        RoleArn: !GetAtt EventBridgeCrossAccountRole.Arn

EventBridgeCrossAccountRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: events.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: PutEventsToCentralBus
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action: events:PutEvents
              Resource: arn:aws:events:us-east-1:SECURITY_ACCOUNT_ID:event-bus/health-central
```

### Xử Lý Centralized tại Security Account

```python
import boto3
import json
import os

def handler(event, context):
    """
    Nhận Health events từ nhiều accounts và route đến đúng team.
    """
    source_account = event.get('account')
    detail = event.get('detail', {})
    service = detail.get('service')
    category = detail.get('eventTypeCategory')

    # Map account ID → team
    account_to_team = {
        '111111111111': 'platform-team',
        '222222222222': 'app-team-a',
        '333333333333': 'app-team-b'
    }
    team = account_to_team.get(source_account, 'ops-team')

    # Route đến đúng SNS topic theo team
    sns = boto3.client('sns')
    topic_arn = os.environ.get(f'SNS_TOPIC_{team.upper().replace("-", "_")}')

    if topic_arn:
        sns.publish(
            TopicArn=topic_arn,
            Subject=f"[{source_account}] AWS Health: {service} {category}",
            Message=json.dumps(event, default=str, indent=2)
        )

    # Log tất cả events vào central CloudWatch
    import time
    logs = boto3.client('logs')
    log_group = '/aws/health/centralized'

    try:
        logs.put_log_events(
            logGroupName=log_group,
            logStreamName=f"{source_account}/{service}",
            logEvents=[{
                'timestamp': int(time.time() * 1000),
                'message': json.dumps(event, default=str)
            }]
        )
    except Exception as e:
        print(f"Warning: Could not write to CloudWatch Logs: {e}")

    return {'routed_to': team, 'account': source_account}
```

---

## Step Functions cho Complex Workflows

Khi workflow phức tạp hơn (cần retry, điều kiện phân nhánh, chờ approval), dùng **Step Functions** (Máy Trạng Thái — State Machine) thay vì Lambda đơn thuần.

### State Machine: EC2 Retirement với Approval

```json
{
  "Comment": "EC2 Instance Retirement Workflow with Human Approval",
  "StartAt": "CheckEnvironment",
  "States": {
    "CheckEnvironment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:check-env",
      "Next": "RouteByEnvironment"
    },
    "RouteByEnvironment": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.environment",
          "StringEquals": "production",
          "Next": "WaitForApproval"
        },
        {
          "Variable": "$.environment",
          "StringEquals": "staging",
          "Next": "ScheduleMigration"
        }
      ],
      "Default": "AutoMigrate"
    },
    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Parameters": {
        "FunctionName": "send-approval-request",
        "Payload": {
          "taskToken.$": "$$.Task.Token",
          "instance.$": "$.instanceId",
          "message": "Production EC2 instance needs retirement migration. Approve?"
        }
      },
      "TimeoutSeconds": 86400,
      "HeartbeatSeconds": 3600,
      "Next": "AutoMigrate"
    },
    "ScheduleMigration": {
      "Type": "Wait",
      "Seconds": 86400,
      "Next": "AutoMigrate"
    },
    "AutoMigrate": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:migrate-ec2",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Next": "VerifyMigration"
    },
    "VerifyMigration": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:verify-migration",
      "Next": "NotifySuccess",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "NotifyFailure"
        }
      ]
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:REGION:ACCOUNT:ops-alerts",
        "Message.$": "States.Format('✅ Migration successful: {}', $.instanceId)"
      },
      "End": true
    },
    "NotifyFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:REGION:ACCOUNT:ops-alerts",
        "Message.$": "States.Format('❌ Migration FAILED: {}. Manual intervention required.', $.instanceId)"
      },
      "End": true
    }
  }
}
```

---

## Monitoring & Observability của Pipeline

### Metrics Cần Theo Dõi

```
CloudWatch Metrics cho Health Automation Pipeline:
├── EventBridge
│   ├── MatchedEvents — số events bắt được
│   ├── TriggeredRules — số rules kích hoạt
│   └── FailedInvocations — số lần fail khi gọi target
│
├── Lambda
│   ├── Invocations — số lần gọi
│   ├── Errors — số lỗi
│   ├── Duration — thời gian xử lý
│   └── Throttles — số lần bị throttle
│
└── Step Functions (nếu dùng)
    ├── ExecutionsStarted
    ├── ExecutionsFailed
    ├── ExecutionTime
    └── ExecutionsTimedOut
```

### CloudWatch Dashboard cho Health Pipeline

```yaml
HealthPipelineDashboard:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: aws-health-automation
    DashboardBody: !Sub |
      {
        "widgets": [
          {
            "type": "metric",
            "properties": {
              "title": "Health Events Processed (24h)",
              "metrics": [
                ["AWS/Events", "MatchedEvents", "RuleName", "aws-health-all-events"],
                ["AWS/Lambda", "Invocations", "FunctionName", "health-processor"],
                ["AWS/Lambda", "Errors", "FunctionName", "health-processor"]
              ],
              "period": 3600,
              "stat": "Sum",
              "view": "timeSeries"
            }
          }
        ]
      }
```

### CloudWatch Alarm cho Pipeline Failures

```yaml
LambdaErrorAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: health-processor-errors
    AlarmDescription: Health event processor Lambda đang có lỗi
    MetricName: Errors
    Namespace: AWS/Lambda
    Dimensions:
      - Name: FunctionName
        Value: health-processor
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    AlarmActions:
      - !Ref AlertTopic
    TreatMissingData: notBreaching
```

---

## Infrastructure as Code với Terraform

### Module hoàn chỉnh với Terraform

```hcl
# variables.tf
variable "slack_webhook_url" {
  description = "Slack Incoming Webhook URL để gửi alerts"
  type        = string
  sensitive   = true
}

variable "alert_email" {
  description = "Email nhận Health alerts"
  type        = string
}

variable "environment" {
  description = "Tên môi trường (dev/staging/production)"
  type        = string
  default     = "production"
}

# main.tf
locals {
  name_prefix = "aws-health-${var.environment}"
}

# SNS Topic cho alerts
resource "aws_sns_topic" "health_alerts" {
  name = "${local.name_prefix}-alerts"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.health_alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# SSM Parameter cho Slack webhook
resource "aws_ssm_parameter" "slack_webhook" {
  name  = "/${local.name_prefix}/slack-webhook-url"
  type  = "SecureString"
  value = var.slack_webhook_url
}

# Lambda function
resource "aws_lambda_function" "health_processor" {
  filename         = "health_processor.zip"
  function_name    = "${local.name_prefix}-processor"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "python3.12"
  timeout          = 60

  environment {
    variables = {
      ALERT_TOPIC_ARN          = aws_sns_topic.health_alerts.arn
      SLACK_WEBHOOK_PARAM_NAME = aws_ssm_parameter.slack_webhook.name
      ENVIRONMENT              = var.environment
    }
  }
}

# IAM Role cho Lambda
resource "aws_iam_role" "lambda_role" {
  name = "${local.name_prefix}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "lambda_policy" {
  role = aws_iam_role.lambda_role.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["sns:Publish"]
        Resource = aws_sns_topic.health_alerts.arn
      },
      {
        Effect = "Allow"
        Action = ["ssm:GetParameter"]
        Resource = aws_ssm_parameter.slack_webhook.arn
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeInstances",
          "ec2:StopInstances",
          "ec2:StartInstances",
          "ec2:CreateTags"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      }
    ]
  })
}

# EventBridge Rule
resource "aws_cloudwatch_event_rule" "health_events" {
  name        = "${local.name_prefix}-all-events"
  description = "Capture AWS Health events for ${var.environment}"

  event_pattern = jsonencode({
    source      = ["aws.health"]
    detail-type = ["AWS Health Event"]
    detail = {
      eventTypeCategory = ["issue", "scheduledChange"]
    }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.health_events.name
  target_id = "ProcessWithLambda"
  arn       = aws_lambda_function.health_processor.arn
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.health_processor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.health_events.arn
}
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao dùng EventBridge thay vì polling Health API?

**Trả lời:** EventBridge là event-driven (hướng sự kiện), có ba lợi thế chính:
1. **Độ trễ thấp** — Phản hồi trong vài giây, polling thường 1–5 phút
2. **Chi phí thấp** — Không tốn tiền gọi API định kỳ; chỉ trả tiền khi có event
3. **Đơn giản hơn** — Không cần quản lý scheduler, không cần xử lý pagination

---

### Q2: Thiết kế hệ thống tự động phản hồi Health events cho 50 AWS accounts?

**Trả lời:** Kiến trúc hub-and-spoke tập trung:
1. Mỗi member account có EventBridge Rule forward Health events sang **Central Event Bus** tại Security/Operations account
2. Central account có Lambda xử lý, phân loại theo account/service/severity
3. Route đến đúng team qua SNS topic riêng cho từng team
4. Log tất cả events vào CloudWatch Logs tập trung
5. Dùng Organizations integration để xem organizational view trong Health Console

---

### Q3: Làm sao test EventBridge rule cho Health events mà không cần đợi sự cố thực tế?

**Trả lời:** Dùng `aws events put-events` để inject event giả:

```bash
aws events put-events --entries '[
  {
    "Source": "aws.health",
    "DetailType": "AWS Health Event",
    "Detail": "{\"service\":\"EC2\",\"eventTypeCode\":\"AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED\",\"eventTypeCategory\":\"scheduledChange\",\"affectedEntities\":[{\"entityValue\":\"i-0test123\"}]}",
    "EventBusName": "default"
  }
]'
```

> **Lưu ý:** AWS Health events thực sự không thể giả lập hoàn toàn vì có các trường internal đặc biệt. Dùng approach này cho unit testing, kết hợp với mock trong Lambda test cho integration testing.

---

### Q4: Sự khác biệt giữa target Lambda và target SNS trong EventBridge rule là gì?

**Trả lời:**

| Target        | Khi nào dùng                                                  |
| ------------- | ------------------------------------------------------------- |
| **Lambda**    | Cần xử lý logic phức tạp, transform data, gọi multiple APIs  |
| **SNS**       | Fan-out đơn giản, notify nhiều subscriber cùng lúc            |
| **SQS**       | Buffer events, xử lý async, retry tự động                    |
| **Step Functions** | Workflow phức tạp, có điều kiện, cần retry và timeout   |

---

**Điều Hướng:**
- ← [1-personal-health.md — PHD chi tiết](./1-personal-health.md)
- ← [README.md — Tổng quan Health Dashboard](./README.md)
- → [../08-trusted-advisor/ — Trusted Advisor](../08-trusted-advisor/README.md)

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
