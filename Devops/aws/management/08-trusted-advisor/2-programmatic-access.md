# Trusted Advisor — Programmatic Access, EventBridge & Automation

> Hướng dẫn chi tiết truy cập Trusted Advisor qua **Support API** (Giao Diện Lập Trình Dịch Vụ Hỗ Trợ), tích hợp với **EventBridge** (Cầu Nối Sự Kiện) để tự động hóa phản ứng, và xây dựng **automation workflows** (Luồng Tự Động Hóa) phản ứng với checks thay đổi trạng thái.

---

## 📚 Mục Lục

1. [Tổng Quan Programmatic Access](#tổng-quan-programmatic-access)
2. [Support API — Giao Diện Lập Trình](#support-api--giao-diện-lập-trình)
3. [EventBridge Integration](#eventbridge-integration)
4. [Automation Patterns — Mẫu Tự Động Hóa](#automation-patterns--mẫu-tự-động-hóa)
5. [Multi-Account Trusted Advisor](#multi-account-trusted-advisor)
6. [Dashboard & Reporting Tùy Chỉnh](#dashboard--reporting-tùy-chỉnh)
7. [Giới Hạn & Best Practices](#giới-hạn--best-practices)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Programmatic Access

### Ba Phương Thức Truy Cập Trusted Advisor

```
┌──────────────────────────────────────────────────────────────────────┐
│                   Trusted Advisor Access Methods                      │
│                                                                      │
│  1. Console UI (Giao Diện Đồ Họa)                                   │
│     URL: console.aws.amazon.com/trustedadvisor                       │
│     Phù hợp: Xem thủ công, không tự động hóa được                   │
│                                                                      │
│  2. Support API (Giao Diện Lập Trình qua AWS SDK)                   │
│     Endpoint: support.us-east-1.amazonaws.com (LUÔN us-east-1)      │
│     Phù hợp: Pull results, integrate vào dashboard, CI/CD           │
│     Yêu cầu: Business hoặc Enterprise Support plan                   │
│                                                                      │
│  3. EventBridge Events (Sự Kiện Tự Động)                            │
│     Trigger: Khi check status thay đổi (OK→WARNING→ERROR)           │
│     Phù hợp: Real-time automation, không cần polling                 │
│     Yêu cầu: Business hoặc Enterprise Support plan                   │
└──────────────────────────────────────────────────────────────────────┘
```

### Điều Kiện Tiên Quyết

```
✅ Support Plan: Business ($100+/tháng) hoặc Enterprise ($15,000+/tháng)
✅ IAM Permissions cần thiết:
   - support:DescribeTrustedAdvisorChecks
   - support:DescribeTrustedAdvisorCheckResult
   - support:DescribeTrustedAdvisorCheckSummaries
   - support:RefreshTrustedAdvisorCheck

✅ Region: PHẢI dùng us-east-1 cho Support API
   (Dù tài nguyên ở region khác, API endpoint vẫn là us-east-1)
```

---

## Support API — Giao Diện Lập Trình

### Các Operations Chính

```
┌─────────────────────────────────────────────────────────────────┐
│                    Support API Operations                        │
│                                                                 │
│  DescribeTrustedAdvisorChecks                                   │
│  └─ Liệt kê tất cả checks và metadata                         │
│                                                                 │
│  DescribeTrustedAdvisorCheckResult                              │
│  └─ Kết quả chi tiết của 1 check cụ thể                       │
│                                                                 │
│  DescribeTrustedAdvisorCheckSummaries                           │
│  └─ Tóm tắt nhiều checks cùng lúc (OK/WARNING/ERROR count)    │
│                                                                 │
│  RefreshTrustedAdvisorCheck                                     │
│  └─ Yêu cầu refresh on-demand (rate limited)                   │
│                                                                 │
│  DescribeTrustedAdvisorCheckRefreshStatuses                     │
│  └─ Kiểm tra trạng thái refresh đang chạy                     │
└─────────────────────────────────────────────────────────────────┘
```

### Python SDK — Ví Dụ Đầy Đủ

```python
import boto3
import json
from datetime import datetime

# QUAN TRỌNG: region_name PHẢI là 'us-east-1'
# Dù account của bạn ở ap-southeast-1 hay eu-west-1
client = boto3.client('support', region_name='us-east-1')

# ─────────────────────────────────────────────────
# 1. Liệt kê tất cả Trusted Advisor checks
# ─────────────────────────────────────────────────
def list_all_checks():
    response = client.describe_trusted_advisor_checks(language='en')
    checks = response['checks']

    # Nhóm theo category
    by_category = {}
    for check in checks:
        cat = check['category']
        if cat not in by_category:
            by_category[cat] = []
        by_category[cat].append({
            'id': check['id'],
            'name': check['name'],
            'description': check['description'][:100]
        })

    for category, check_list in by_category.items():
        print(f"\n=== {category.upper()} ({len(check_list)} checks) ===")
        for c in check_list:
            print(f"  [{c['id']}] {c['name']}")

    return checks

# ─────────────────────────────────────────────────
# 2. Lấy kết quả chi tiết của một check
# ─────────────────────────────────────────────────
def get_check_result(check_id: str):
    response = client.describe_trusted_advisor_check_result(
        checkId=check_id,
        language='en'
    )
    result = response['result']

    print(f"Check Status: {result['status']}")  # ok | warning | error | not_available
    print(f"Timestamp: {result['timestamp']}")

    # Thống kê resources
    resources = result.get('flaggedResources', [])
    print(f"Flagged Resources: {len(resources)}")

    for resource in resources[:5]:  # In 5 resource đầu tiên
        print(f"  - Status: {resource['status']}, Region: {resource.get('region', 'N/A')}")
        print(f"    Metadata: {resource.get('metadata', [])}")

    return result

# ─────────────────────────────────────────────────
# 3. Quét toàn bộ checks và lọc theo status
# ─────────────────────────────────────────────────
def get_all_errors_and_warnings():
    # Bước 1: Lấy danh sách check IDs
    checks = client.describe_trusted_advisor_checks(language='en')['checks']
    check_ids = [c['id'] for c in checks]

    # Bước 2: Lấy summaries (batch, tối đa ~300 checks cùng lúc)
    summaries = client.describe_trusted_advisor_check_summaries(
        checkIds=check_ids
    )['summaries']

    # Bước 3: Lọc issues
    issues = []
    for summary in summaries:
        if summary['status'] in ('warning', 'error'):
            issues.append({
                'checkId': summary['checkId'],
                'status': summary['status'],
                'timestamp': summary['timestamp'],
                'resourcesSummary': summary['resourcesSummary']
            })

    # Sắp xếp: error trước, warning sau
    issues.sort(key=lambda x: (0 if x['status'] == 'error' else 1))

    print(f"\nTổng cộng {len(issues)} vấn đề cần xử lý:")
    for issue in issues:
        emoji = "❌" if issue['status'] == 'error' else "⚠️"
        resources_flagged = issue['resourcesSummary'].get('resourcesFlagged', 0)
        print(f"{emoji} [{issue['status'].upper()}] CheckID: {issue['checkId']}"
              f" — {resources_flagged} resources")

    return issues

# ─────────────────────────────────────────────────
# 4. Refresh một check on-demand
# ─────────────────────────────────────────────────
def refresh_check(check_id: str):
    try:
        response = client.refresh_trusted_advisor_check(checkId=check_id)
        status = response['status']
        print(f"Refresh status: {status['status']}")
        print(f"Milliseconds until next refresh: {status.get('millisUntilNextRefreshable', 0)}")
    except client.exceptions.ThrottlingException:
        print("Rate limit reached — Trusted Advisor refresh has a cooldown period")

# ─────────────────────────────────────────────────
# 5. Check cụ thể: MFA on Root Account
# ─────────────────────────────────────────────────
WELL_KNOWN_CHECK_IDS = {
    'mfa_root_account': 'Pfx0RwqBli',
    'security_groups_unrestricted': 'HCP4007jGY',
    's3_bucket_permissions': 'Pfx0RwqBli',
    'cloudtrail_logging': 'vjafUGJ9H0',
    'rds_multi_az': 'f2iK5R6Dep',
    'iam_access_key_rotation': 'DqdJqYeRm5',
    'ebs_snapshots': 'H7IgTzjTYb',
}

result = get_check_result(WELL_KNOWN_CHECK_IDS['mfa_root_account'])
```

### IAM Policy Tối Thiểu Cần Thiết

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TrustedAdvisorReadOnly",
      "Effect": "Allow",
      "Action": [
        "support:DescribeTrustedAdvisorChecks",
        "support:DescribeTrustedAdvisorCheckResult",
        "support:DescribeTrustedAdvisorCheckSummaries",
        "support:DescribeTrustedAdvisorCheckRefreshStatuses"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TrustedAdvisorRefresh",
      "Effect": "Allow",
      "Action": [
        "support:RefreshTrustedAdvisorCheck"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## EventBridge Integration

### Cấu Trúc Event Trusted Advisor Gửi Đến EventBridge

Khi trạng thái của một check thay đổi, Trusted Advisor tự động phát event:

```json
{
  "version": "0",
  "id": "event-uuid-example",
  "source": "aws.trustedadvisor",
  "account": "123456789012",
  "time": "2026-05-17T10:30:00Z",
  "region": "us-east-1",
  "detail-type": "Trusted Advisor Check Item Refresh Notification",
  "detail": {
    "check-name": "Security Groups - Unrestricted Access",
    "check-item-detail": {
      "Status": "error",
      "Region": "ap-southeast-1",
      "Protocol": "tcp",
      "Port": "22",
      "IP Range": "0.0.0.0/0",
      "Resource": "sg-0abc123def456789"
    },
    "status": "ERROR",
    "resource_id": "sg-0abc123def456789",
    "uuid": "check-item-uuid"
  }
}
```

**Các giá trị `status` trong event:**
```
"OK"            — Check về trạng thái bình thường (vấn đề đã được fix)
"WARNING"       — Check phát hiện vấn đề mức cảnh báo
"ERROR"         — Check phát hiện vấn đề nghiêm trọng
"NOT_AVAILABLE" — Check không thể thực thi
```

### EventBridge Rule — Bắt Tất Cả Trusted Advisor Events

```json
{
  "source": ["aws.trustedadvisor"],
  "detail-type": ["Trusted Advisor Check Item Refresh Notification"]
}
```

### EventBridge Rule — Bắt Chỉ ERROR Status

```json
{
  "source": ["aws.trustedadvisor"],
  "detail-type": ["Trusted Advisor Check Item Refresh Notification"],
  "detail": {
    "status": ["ERROR"]
  }
}
```

### EventBridge Rule — Bắt Security Check Cụ Thể

```json
{
  "source": ["aws.trustedadvisor"],
  "detail-type": ["Trusted Advisor Check Item Refresh Notification"],
  "detail": {
    "check-name": [
      "Security Groups - Unrestricted Access",
      "Amazon S3 Bucket Permissions",
      "MFA on Root Account",
      "AWS CloudTrail Logging"
    ],
    "status": ["ERROR", "WARNING"]
  }
}
```

---

## Automation Patterns — Mẫu Tự Động Hóa

### Pattern 1: Alert to Slack khi phát hiện Security Issue

```
Trusted Advisor phát hiện Security Group 0.0.0.0/0:22
        │
        ▼
EventBridge Rule (filter: check-name contains "Security Groups", status=ERROR)
        │
        ▼
Lambda Function "ta-security-alerter"
        │
        ├─── Parse event: lấy resource_id, region, port, IP range
        ├─── Gửi Slack message: "@channel ⚠️ Security Group sg-xxx đang mở port 22 toàn internet!"
        └─── Tạo AWS Support Case hoặc Jira ticket
```

**Lambda code (Python):**
```python
import boto3
import json
import urllib.request

SLACK_WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

def lambda_handler(event, context):
    detail = event['detail']
    check_name = detail['check-name']
    status = detail['status']
    check_detail = detail.get('check-item-detail', {})
    resource_id = detail.get('resource_id', 'unknown')
    region = check_detail.get('Region', 'unknown')

    emoji = "🔴" if status == "ERROR" else "🟡"
    message = {
        "text": f"{emoji} *Trusted Advisor Alert*",
        "attachments": [{
            "color": "danger" if status == "ERROR" else "warning",
            "fields": [
                {"title": "Check", "value": check_name, "short": False},
                {"title": "Status", "value": status, "short": True},
                {"title": "Region", "value": region, "short": True},
                {"title": "Resource", "value": resource_id, "short": False},
                {"title": "Details", "value": json.dumps(check_detail, indent=2), "short": False}
            ]
        }]
    }

    data = json.dumps(message).encode('utf-8')
    req = urllib.request.Request(SLACK_WEBHOOK_URL, data=data,
                                  headers={'Content-Type': 'application/json'})
    urllib.request.urlopen(req)

    return {"statusCode": 200}
```

---

### Pattern 2: Auto-Remediate — Tự Động Đóng Security Group

```
Trusted Advisor: Security Group sg-xxx mở 0.0.0.0/0:22 → ERROR
        │
        ▼
EventBridge → Lambda "ta-auto-remediate-sg"
        │
        ▼
Lambda kiểm tra tag: "AutoRemediate" = "true"?
        │
        ├── Có tag → Revoke inbound rule 0.0.0.0/0:22
        │           → Log action vào CloudWatch Logs
        │           → Gửi Slack notification: "Auto-fixed sg-xxx"
        │
        └── Không tag → Chỉ alert, không auto-fix
                       → Tạo OpsItem trên SSM OpsCenter
```

**Lambda code (Python):**
```python
import boto3

ec2 = boto3.client('ec2')
ssm = boto3.client('ssm', region_name='ap-southeast-1')

def lambda_handler(event, context):
    detail = event['detail']
    check_name = detail.get('check-name', '')

    if 'Security Groups' not in check_name:
        return

    check_item = detail.get('check-item-detail', {})
    resource_id = detail.get('resource_id')   # Security Group ID
    region = check_item.get('Region')
    port = int(check_item.get('Port', 0))
    protocol = check_item.get('Protocol', 'tcp')
    ip_range = check_item.get('IP Range', '')

    ec2_regional = boto3.client('ec2', region_name=region)

    # Kiểm tra tag AutoRemediate
    sg_response = ec2_regional.describe_security_groups(GroupIds=[resource_id])
    sg = sg_response['SecurityGroups'][0]
    tags = {t['Key']: t['Value'] for t in sg.get('Tags', [])}

    if tags.get('AutoRemediate') != 'true':
        # Tạo OpsItem thay vì auto-fix
        create_ops_item(resource_id, check_name, region)
        return

    # Auto-fix: Revoke ingress rule
    try:
        if ip_range == '0.0.0.0/0':
            ec2_regional.revoke_security_group_ingress(
                GroupId=resource_id,
                IpPermissions=[{
                    'IpProtocol': protocol,
                    'FromPort': port,
                    'ToPort': port,
                    'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                }]
            )
        print(f"Auto-remediated: Revoked port {port} from {resource_id} in {region}")
    except Exception as e:
        print(f"Remediation failed: {e}")
        create_ops_item(resource_id, check_name, region)


def create_ops_item(resource_id, check_name, region):
    ssm.create_ops_item(
        Title=f"Trusted Advisor: {check_name}",
        Description=f"Resource {resource_id} in {region} violates best practice.",
        Source="TrustedAdvisor",
        OperationalData={
            'ResourceId': {'Value': resource_id, 'Type': 'SearchableString'},
            'CheckName': {'Value': check_name, 'Type': 'SearchableString'},
        },
        Priority=2
    )
```

---

### Pattern 3: Daily Compliance Report — Báo Cáo Tuân Thủ Hàng Ngày

```
EventBridge Scheduler: Mỗi ngày 8:00 AM (Asia/Ho_Chi_Minh)
        │
        ▼
Lambda "ta-daily-report-generator"
        │
        ├─ Gọi Support API: DescribeTrustedAdvisorCheckSummaries
        ├─ Tạo report: tổng số OK/WARNING/ERROR mỗi category
        ├─ So sánh với ngày hôm qua (lấy từ DynamoDB)
        ├─ Lưu snapshot hôm nay vào DynamoDB
        │
        └─ Gửi email qua SES:
           "Daily Trusted Advisor Report — 2026-05-17
            Security: 2 ERROR ↑ (tăng 1 so với hôm qua)
            Cost: 5 WARNING → (giữ nguyên)
            Fault Tolerance: 0 ERROR ✅"
```

**CloudFormation template tạo infrastructure:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  # DynamoDB lưu snapshot hàng ngày
  TrustedAdvisorHistoryTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: trusted-advisor-history
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: date
          AttributeType: S
        - AttributeName: checkId
          AttributeType: S
      KeySchema:
        - AttributeName: date
          KeyType: HASH
        - AttributeName: checkId
          KeyType: RANGE
      TimeToLiveSpecification:
        AttributeName: ttl
        Enabled: true

  # Lambda function
  DailyReportFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ta-daily-report-generator
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt DailyReportRole.Arn
      Timeout: 300
      Environment:
        Variables:
          DYNAMODB_TABLE: trusted-advisor-history
          SES_RECIPIENT: ops-team@company.com

  # EventBridge Scheduler (Lịch Trình Sự Kiện)
  DailyReportSchedule:
    Type: AWS::Scheduler::Schedule
    Properties:
      Name: ta-daily-report
      ScheduleExpression: "cron(0 1 * * ? *)"  # 8:00 AM GMT+7 = 1:00 AM UTC
      ScheduleExpressionTimezone: Asia/Ho_Chi_Minh
      FlexibleTimeWindow:
        Mode: "OFF"
      Target:
        Arn: !GetAtt DailyReportFunction.Arn
        RoleArn: !GetAtt SchedulerRole.Arn

  DailyReportRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: TrustedAdvisorAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - support:DescribeTrustedAdvisorChecks
                  - support:DescribeTrustedAdvisorCheckSummaries
                  - dynamodb:PutItem
                  - dynamodb:GetItem
                  - ses:SendEmail
                Resource: "*"

  SchedulerRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: scheduler.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: InvokeLambda
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: lambda:InvokeFunction
                Resource: !GetAtt DailyReportFunction.Arn
```

---

### Pattern 4: Service Limit Monitor — Theo Dõi Giới Hạn Dịch Vụ

```
Mục tiêu: Phát hiện khi account sắp chạm Service Quota,
tự động request quota increase trước khi bị block.

Flow:
1. EventBridge Scheduler (hàng ngày)
   → Lambda: Lấy Service Limits checks từ Trusted Advisor
   → Lọc checks có status = WARNING (≥80% quota)
   → Với mỗi check WARNING:
      a. Lookup quota code trong mapping table
      b. Gọi Service Quotas API: describe_service_quota
      c. Tính: current_usage = quota × 0.8 (ước tính)
      d. Tạo Support case: request increase quota × 2
   → Gửi Slack: "Tự động request tăng quota cho EC2 vCPUs từ 96 lên 192"
```

---

## Multi-Account Trusted Advisor

### Vấn Đề Với Nhiều Account

```
Không có cơ chế native aggregate Trusted Advisor kết quả
từ nhiều account vào một console (trừ Enterprise Support + Organizations view)

Giải pháp tự xây:

Management Account
  └─ Lambda "ta-aggregator"
       ├─ Assume Role vào Account-A → Get Trusted Advisor results
       ├─ Assume Role vào Account-B → Get Trusted Advisor results
       ├─ Assume Role vào Account-C → Get Trusted Advisor results
       └─ Aggregate → S3 bucket → Athena → QuickSight Dashboard
```

### IAM Role Setup Cho Cross-Account Access

**Trong mỗi member account (lặp lại cho mỗi account):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:role/ta-aggregator-role"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "trusted-advisor-aggregation-2026"
        }
      }
    }
  ]
}
```

**Trong management account:**
```python
import boto3

def get_trusted_advisor_for_account(account_id: str, external_id: str):
    sts = boto3.client('sts')

    # Assume role vào member account
    creds = sts.assume_role(
        RoleArn=f"arn:aws:iam::{account_id}:role/trusted-advisor-reader",
        RoleSessionName="ta-aggregation",
        ExternalId=external_id
    )['Credentials']

    # Tạo Support client với credentials của member account
    support_client = boto3.client(
        'support',
        region_name='us-east-1',
        aws_access_key_id=creds['AccessKeyId'],
        aws_secret_access_key=creds['SecretAccessKey'],
        aws_session_token=creds['SessionToken']
    )

    # Lấy summaries
    checks = support_client.describe_trusted_advisor_checks(language='en')['checks']
    check_ids = [c['id'] for c in checks]
    summaries = support_client.describe_trusted_advisor_check_summaries(
        checkIds=check_ids
    )['summaries']

    return {
        'account_id': account_id,
        'summaries': summaries,
        'timestamp': datetime.utcnow().isoformat()
    }

# Aggregate từ nhiều account
accounts = ['111111111111', '222222222222', '333333333333']
all_results = []
for account_id in accounts:
    result = get_trusted_advisor_for_account(account_id, 'trusted-advisor-aggregation-2026')
    all_results.append(result)

# Lưu vào S3
s3 = boto3.client('s3')
s3.put_object(
    Bucket='ta-reports-bucket',
    Key=f"aggregated/{datetime.utcnow().strftime('%Y/%m/%d')}/report.json",
    Body=json.dumps(all_results)
)
```

---

## Dashboard & Reporting Tùy Chỉnh

### Architecture: Trusted Advisor Dashboard Tùy Chỉnh

```
┌───────────────────────────────────────────────────────────────────┐
│                   Custom TA Dashboard Architecture                 │
│                                                                   │
│  Data Collection Layer (Thu Thập Dữ Liệu):                       │
│    EventBridge Scheduler (hàng ngày/giờ)                         │
│    → Lambda ta-collector                                          │
│    → Support API (us-east-1)                                      │
│    → S3 (Parquet format, partitioned by date/account/category)    │
│                                                                   │
│  Processing Layer (Xử Lý):                                       │
│    S3 → AWS Glue Crawler → Glue Data Catalog                     │
│    → Amazon Athena (SQL queries trên S3)                          │
│                                                                   │
│  Visualization Layer (Trực Quan Hóa):                            │
│    Athena → Amazon QuickSight Dashboard                           │
│    Hoặc: S3 → Grafana (qua Athena datasource)                    │
│                                                                   │
│  Alerting Layer (Cảnh Báo):                                      │
│    EventBridge (real-time) → SNS → Slack/PagerDuty/Email         │
└───────────────────────────────────────────────────────────────────┘
```

### CloudWatch Dashboard — Theo Dõi Trusted Advisor Metrics

AWS không export Trusted Advisor metrics sang CloudWatch tự động, nhưng bạn có thể tự push:

```python
cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

def push_ta_metrics_to_cloudwatch(account_id: str, summaries: list):
    metric_data = []

    # Count theo status và category
    counts = {'cost': {}, 'performance': {}, 'security': {}, 'fault_tolerant': {}, 'service_limits': {}}

    for summary in summaries:
        # Cần lookup category từ check ID (cần mapping riêng)
        status = summary['status']  # ok | warning | error
        category = 'security'  # simplified

        if status not in counts[category]:
            counts[category][status] = 0
        counts[category][status] += 1

    # Đẩy metrics
    for category, status_counts in counts.items():
        for status, count in status_counts.items():
            metric_data.append({
                'MetricName': 'CheckCount',
                'Dimensions': [
                    {'Name': 'AccountId', 'Value': account_id},
                    {'Name': 'Category', 'Value': category},
                    {'Name': 'Status', 'Value': status}
                ],
                'Value': count,
                'Unit': 'Count'
            })

    cloudwatch.put_metric_data(
        Namespace='Custom/TrustedAdvisor',
        MetricData=metric_data
    )
```

---

## Giới Hạn & Best Practices

### Giới Hạn Kỹ Thuật

```
┌─────────────────────────────────────────────────────────────────┐
│                     Giới Hạn Quan Trọng                         │
│                                                                 │
│ Refresh Rate:                                                   │
│   Auto-refresh: 1 lần/tuần (thứ Tư UTC)                       │
│   On-demand: 1 lần/5 phút mỗi check (rate limited)            │
│                                                                 │
│ API Rate Limits (Throttling Limits):                           │
│   DescribeTrustedAdvisorChecks: 1 lần/giây                    │
│   DescribeTrustedAdvisorCheckResult: 3 lần/giây               │
│   RefreshTrustedAdvisorCheck: 1 lần/5 phút mỗi check         │
│                                                                 │
│ Regional Restriction (Giới Hạn Region):                        │
│   Support API: CHỈ us-east-1                                   │
│   EventBridge events: Được phát ở us-east-1                   │
│                                                                 │
│ Check Coverage (Phạm Vi Kiểm Tra):                             │
│   Không kiểm tra được: Cấu hình bên trong OS/application      │
│   Không kiểm tra được: EKS/ECS workloads chi tiết             │
│   Không kiểm tra được: Custom resources hoặc third-party      │
└─────────────────────────────────────────────────────────────────┘
```

### Best Practices — Thực Hành Tốt Nhất

**1. Không polling liên tục:**
```
❌ SAI: Lambda chạy mỗi phút, gọi Support API
✅ ĐÚNG: Dùng EventBridge events (push) thay vì polling (pull)
         EventBridge → Lambda khi có thay đổi
         Chỉ dùng API để lấy full snapshot hàng ngày
```

**2. Retry với exponential backoff:**
```python
import time
import random

def describe_check_with_retry(check_id: str, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            return client.describe_trusted_advisor_check_result(
                checkId=check_id, language='en'
            )
        except client.exceptions.ThrottlingException:
            if attempt == max_retries - 1:
                raise
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait_time)
```

**3. Cache kết quả:**
```
Kết quả check chỉ refresh tối đa 1 lần/5 phút
→ Cache trong Lambda memory, DynamoDB, hoặc ElastiCache
→ Không gọi lại API nếu data chưa cũ hơn 5 phút
```

**4. Gắn tag tài nguyên để auto-remediation an toàn:**
```
Tag "AutoRemediate" = "true"  → Lambda tự động fix
Tag "AutoRemediate" = "false" → Chỉ alert, không fix
Tag không có               → Chỉ alert (default an toàn)
```

**5. Tách Lambda theo responsibility:**
```
Lambda 1: ta-event-processor — Nhận EventBridge events, route sang Lambda khác
Lambda 2: ta-security-remediation — Xử lý Security checks
Lambda 3: ta-cost-reporter — Xử lý Cost Optimization checks
Lambda 4: ta-limit-monitor — Xử lý Service Limits checks
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Làm sao đọc kết quả Trusted Advisor bằng code?

**Trả lời:**
Dùng AWS Support API với `boto3.client('support', region_name='us-east-1')`. Bắt buộc phải dùng `us-east-1` vì Support API là global endpoint đặt ở đó. Các operations chính:
- `describe_trusted_advisor_checks()` — Lấy danh sách tất cả checks
- `describe_trusted_advisor_check_result(checkId=...)` — Kết quả chi tiết một check
- `refresh_trusted_advisor_check(checkId=...)` — Refresh on-demand (Business+ Support)

### Q2: Làm sao tự động phản ứng khi Trusted Advisor phát hiện Security Group 0.0.0.0/0?

**Trả lời:**
1. Trusted Advisor phát EventBridge event khi check status thay đổi
2. Tạo EventBridge Rule filter `source: aws.trustedadvisor` và `check-name: "Security Groups - Unrestricted Access"`
3. Lambda nhận event, parse Security Group ID từ `detail.resource_id`
4. Lambda kiểm tra tag `AutoRemediate` — nếu có, gọi `ec2.revoke_security_group_ingress()`
5. Gửi Slack alert và tạo OpsItem trên SSM OpsCenter để theo dõi

### Q3: Làm sao aggregate Trusted Advisor results từ nhiều AWS account?

**Trả lời:**
Không có cơ chế native (trừ Enterprise Support với Organizations view). Cách tự xây:
1. Tạo IAM Role trong mỗi member account cho phép management account assume
2. Lambda trong management account assume role vào từng member account
3. Gọi Support API (us-east-1) với credentials của từng account
4. Aggregate kết quả vào S3, dùng Athena + QuickSight để visualize

### Q4: EventBridge rule bắt Trusted Advisor events phải đặt ở region nào?

**Trả lời:**
EventBridge rule phải đặt ở `us-east-1` — cùng region với Support API endpoint. Các events từ Trusted Advisor được publish vào us-east-1 event bus. Nếu muốn trigger resources ở region khác (ví dụ Lambda ở ap-southeast-1), có thể dùng EventBridge cross-region routing hoặc SNS fanout.

### Q5: Phân biệt on-demand refresh và auto-refresh của Trusted Advisor?

**Trả lời:**
- **Auto-refresh**: Tự động mỗi tuần (khoảng thứ Tư UTC), không cần làm gì
- **On-demand refresh**: Gọi `RefreshTrustedAdvisorCheck` qua Support API, chỉ có với Business/Enterprise Support, rate limit 1 lần/5 phút mỗi check. Dùng khi cần xác nhận kết quả ngay sau khi fix vấn đề (VD: Đã đóng Security Group, refresh để xác nhận status về OK)

---

## Điều Hướng

| Điều Hướng                       | Link                                                             |
| --------------------------------- | ---------------------------------------------------------------- |
| ← Trước: Check Categories         | [1-check-categories.md](./1-check-categories.md)                 |
| ← README                          | [README.md](./README.md)                                         |
| → Tiếp: Health Dashboard          | [../09-health-dashboard/README.md](../09-health-dashboard/README.md) |
| ↑ Chỉ Mục                         | [INDEX.md](../INDEX.md)                                          |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
