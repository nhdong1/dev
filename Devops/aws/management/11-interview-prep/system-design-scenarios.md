# Tình Huống Thiết Kế Hệ Thống — AWS Management & Governance

> 6 tình huống thực tế với yêu cầu, kiến trúc đề xuất, diagram và các điểm trade-off cần thảo luận. Luyện tập bằng cách vẽ diagram trên giấy/whiteboard trước khi nhìn đáp án.

---

## Cách Sử Dụng Hiệu Quả

```
1. Đọc yêu cầu (Requirements)
2. Dừng lại — vẽ diagram, liệt kê components (15–20 phút)
3. Đọc đáp án đề xuất
4. So sánh approach của bạn và đáp án
5. Luyện giải thích bằng miệng (3–5 phút)
```

---

## Scenario S1: Observability Platform cho Microservices

### Yêu Cầu

**Bối cảnh:** Công ty SaaS có 20 microservices trên ECS Fargate, 3 region (us-east-1, eu-west-1, ap-southeast-1), 2 account (prod và staging).

**Requirements:**
- R1: Phát hiện sự cố trong < 5 phút
- R2: Dashboard tổng hợp metrics tất cả services
- R3: Không mất log khi service crash
- R4: Alert trên Slack và PagerDuty khi critical
- R5: Query log nhanh để debug (< 30 giây)
- R6: Chi phí monitoring không vượt 5% tổng AWS bill

**Constraints:**
- Không muốn quản lý Prometheus/Grafana server
- Team nhỏ (3 DevOps engineers)

---

### Kiến Trúc Đề Xuất

```
┌─────────────────────────────────────────────────────────────┐
│                        ECS Fargate Services                  │
│  [Service A]  [Service B]  [Service C]  ...  [Service N]    │
│      │              │            │                │          │
│  stdout/stderr (CloudWatch Logs — tự động thu thập)         │
│      │              │            │                │          │
└──────┼──────────────┼────────────┼────────────────┼──────────┘
       │              │            │                │
       ▼              ▼            ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                   CloudWatch Logs                            │
│  Log Group: /ecs/service-a  /ecs/service-b  ...             │
│                                                             │
│  Metric Filters:                                            │
│    - ErrorCount: filter pattern [ERROR]                     │
│    - SlowRequest: filter duration > 2000ms                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
              ┌────────────────┼───────────────┐
              ▼                ▼               ▼
   ┌──────────────────┐  ┌──────────┐  ┌─────────────────┐
   │ CloudWatch       │  │ Logs     │  │ Subscription    │
   │ Metrics          │  │ Insights │  │ Filter → S3     │
   │ (từ Filters)     │  │ (Query)  │  │ (Long-term)     │
   └────────┬─────────┘  └──────────┘  └─────────────────┘
            │
   ┌────────▼─────────────────────────────────────────┐
   │              CloudWatch Alarms                    │
   │                                                  │
   │  Per Service Alarms:                             │
   │    - ErrorRate > 1% (5 phút)                    │
   │    - P95 Latency > 2s (5 phút)                 │
   │    - TaskCount < MinCount (immediate)            │
   │                                                  │
   │  Composite Alarms:                               │
   │    - CriticalAlert = (ErrorRate OR Latency)     │
   │                      AND TaskRunning             │
   └────────────────────┬─────────────────────────────┘
                        │
            ┌───────────▼────────────┐
            │      SNS Topics        │
            │                        │
            │  - critical-alerts    │
            │  - warning-alerts     │
            └──┬────────────────┬───┘
               │                │
        ┌──────▼──┐     ┌──────▼──────────┐
        │  Lambda  │     │  AWS Chatbot    │
        │  (route) │     │  (Slack notify) │
        └──┬───────┘     └─────────────────┘
           │
    ┌──────▼──────┐
    │  PagerDuty  │
    │  (on-call)  │
    └─────────────┘

CloudWatch Dashboard:
┌─────────────────────────────────────────────────────┐
│  Cross-Account, Cross-Region Dashboard              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │Service A │ │Service B │ │Service C │  ...       │
│  │ 🟢 OK    │ │ 🔴 ALARM │ │ 🟢 OK    │           │
│  │Err: 0.1% │ │Err: 5.2% │ │Err: 0.0% │           │
│  │P95: 450ms│ │P95: 3.1s │ │P95: 230ms│           │
│  └──────────┘ └──────────┘ └──────────┘           │
└─────────────────────────────────────────────────────┘
```

### Container Insights Setup

```bash
# Bật Container Insights cho ECS cluster
aws ecs put-account-setting \
  --name containerInsights \
  --value enabled

# Deploy với CloudFormation
ECSCluster:
  Type: AWS::ECS::Cluster
  Properties:
    ClusterSettings:
      - Name: containerInsights
        Value: enabled
```

### CloudFormation Template — Standard Alarm Set Per Service

```yaml
Parameters:
  ServiceName:
    Type: String

Resources:
  ErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${ServiceName}-error-rate'
      MetricName: ErrorCount
      Namespace: !Sub 'ECS/${ServiceName}'
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref CriticalSnsTopic
      OKActions:
        - !Ref CriticalSnsTopic

  LatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${ServiceName}-p95-latency'
      ExtendedStatistic: p95
      MetricName: TargetResponseTime
      Namespace: AWS/ApplicationELB
      Period: 300
      EvaluationPeriods: 2
      Threshold: 2
      ComparisonOperator: GreaterThanThreshold

  CompositeAlarm:
    Type: AWS::CloudWatch::CompositeAlarm
    Properties:
      AlarmName: !Sub '${ServiceName}-critical'
      AlarmRule: !Sub |
        ALARM("${ServiceName}-error-rate") 
        OR ALARM("${ServiceName}-p95-latency")
```

### Trade-offs Cần Thảo Luận

| Quyết Định                        | Option A                      | Option B                       | Chọn & Lý Do               |
| --------------------------------- | ----------------------------- | ------------------------------ | --------------------------- |
| Centralized logging               | CloudWatch Logs               | OpenSearch + Kinesis           | CW Logs — đơn giản hơn     |
| Alerting channel                  | SNS → Email                   | SNS → Chatbot → Slack          | Chatbot vì team dùng Slack  |
| On-call escalation                | SNS → PagerDuty               | SNS → OpsGenie                 | Tùy team preference         |
| Log retention                     | 30 ngày trong CW              | 30 ngày CW + S3 vô thời hạn   | Cả hai — compliance + cost  |
| Custom metrics                    | Metric Filter từ logs         | EMF — Embedded Metric Format   | EMF nếu cần structured      |

---

## Scenario S2: Multi-Account Landing Zone Cho Startup Scale-Up

### Yêu Cầu

**Bối cảnh:** Startup fintech vừa raise Series B, cần migrate từ 1 AWS account sang multi-account architecture trong 3 tháng. Hiện tại: 1 account chứa tất cả dev + staging + prod.

**Requirements:**
- R1: Tách biệt production khỏi dev/staging
- R2: Centralized logging và audit trail
- R3: Single Sign-On cho tất cả developers
- R4: PCI-DSS compliance trong vòng 6 tháng
- R5: Tạo account mới phải hoàn tất trong < 1 giờ
- R6: Không downtime khi migration

**Team:** 2 DevOps engineers, 15 developers, CTO oversees

---

### Kiến Trúc Đề Xuất

```
AWS Organizations
│
├── Management Account (root — chỉ billing, không có workloads)
│   └── Control Tower Dashboard
│
├── OU: Security
│   ├── Log Archive Account
│   │   └── S3: centralized-audit-logs
│   │       ├── cloudtrail/ (Organization Trail logs)
│   │       └── config/ (Config snapshots)
│   └── Audit Account
│       ├── Security Hub (master aggregator)
│       ├── GuardDuty (master aggregator)
│       └── Cross-account readonly roles
│
├── OU: Infrastructure
│   └── Shared Services Account
│       ├── Transit Gateway (hub network)
│       ├── Route 53 (DNS)
│       └── SSO/Identity Center
│
└── OU: Workloads
    ├── OU: Production (SCP: deny Delete* without approval)
    │   ├── Prod-App Account      ← PCI-DSS scope
    │   └── Prod-Data Account     ← PCI-DSS scope
    └── OU: Non-Production (SCP: budget cap, auto-shutdown 22h)
        ├── Dev Account
        └── Staging Account

SCPs Applied:
├── All OUs: deny CloudTrail disable, deny leave Org
├── Production OU: deny rds:Delete, deny s3:DeleteBucket
└── Non-Prod OU: deny m5.4xlarge+, daily cost budget action
```

### Migration Plan (3 Tháng)

```
Tháng 1 — Foundation:
Week 1: Setup Control Tower, tạo Audit, Log Archive accounts
Week 2: Cấu hình IAM Identity Center, import users từ existing IAM
Week 3: Tạo Dev Account, migrate dev workloads
Week 4: Tạo Staging Account, migrate staging workloads

Tháng 2 — Production:
Week 5: Tạo Prod-App Account (sandbox mode — không traffic)
Week 6: Setup networking (Transit Gateway), test connectivity
Week 7: Blue/green: run Prod-App account parallel với production cũ
Week 8: Cutover Prod-App, monitor, rollback ready

Tháng 3 — PCI-DSS Prep:
Week 9:  Tạo Prod-Data Account với enhanced controls
Week 10: Deploy CIS Conformance Pack trên tất cả accounts
Week 11: Security Hub findings remediation
Week 12: Internal audit, prepare evidence
```

### Account Factory Automation (Terraform + AFT)

```hcl
# AFT — Account Factory for Terraform
# Vending machine tự động tạo account mới

module "new_team_account" {
  source = "github.com/aws-ia/terraform-aws-control_tower_account_factory"

  account_name              = "team-payments-prod"
  account_email             = "aws+payments-prod@company.com"
  account_ou                = "OU-Production"
  sso_user_email            = "payments-lead@company.com"

  account_customizations = {
    enable_cloudtrail       = true
    enable_config           = true
    enable_guardduty        = true
    default_region          = "us-east-1"
    additional_regions      = ["eu-west-1"]
    budget_limit_monthly    = 5000
  }
}
```

### Trade-offs Cần Thảo Luận

| Quyết Định                    | Option A                    | Option B                        | Chọn & Lý Do                    |
| ----------------------------- | --------------------------- | ------------------------------- | -------------------------------- |
| Landing zone tool             | Control Tower               | Tự build Organizations          | Control Tower — nhanh hơn 6 tuần |
| IaC cho account               | Account Factory (built-in)  | AFT — Account Factory Terraform | AFT nếu cần Terraform ecosystem  |
| SSO                           | IAM Identity Center         | Okta + SAML federation          | IAM IC nếu không có Okta rồi     |
| Network topology              | Full mesh VPC peering       | Transit Gateway hub             | TGW — scalable hơn              |
| PCI scope isolation           | Separate OU                 | Separate account mỗi service    | Separate account — cleaner scope |

---

## Scenario S3: Compliance-as-Code Pipeline

### Yêu Cầu

**Bối cảnh:** Healthcare company cần đạt HIPAA compliance. Security team phải kiểm tra 500+ tài nguyên mỗi tuần — hiện tại làm thủ công, mất 2 ngày/tuần.

**Requirements:**
- R1: Tự động phát hiện HIPAA violation trong < 15 phút
- R2: Auto-remediate critical violations (unencrypted storage)
- R3: Non-critical violations tạo Jira ticket tự động
- R4: Weekly compliance report gửi email cho CISO
- R5: Audit trail cho mọi remediation action
- R6: Human approval cho remediation ảnh hưởng production

---

### Kiến Trúc Đề Xuất

```
Lớp 1: DETECT (Phát Hiện)
┌─────────────────────────────────────────────────────────────┐
│ AWS Config                                                   │
│                                                             │
│ HIPAA Conformance Pack:                                     │
│   - rds-storage-encrypted              (HIPAA §164.312(a)) │
│   - s3-bucket-server-side-encryption-enabled               │
│   - ebs-encrypted-volumes                                  │
│   - cloudtrail-enabled                                     │
│   - vpc-flow-logs-enabled                                  │
│   - ... (40+ rules)                                        │
└────────────────────────┬────────────────────────────────────┘
                         │ NON_COMPLIANT event
                         ▼
Lớp 2: CLASSIFY (Phân Loại)
┌─────────────────────────────────────────────────────────────┐
│ EventBridge Rule: Config → Lambda Classifier                │
│                                                             │
│ Lambda: classify_violation()                                │
│   Input: Config Rule + Resource Type + Environment         │
│   Output: severity (CRITICAL / HIGH / MEDIUM / LOW)        │
│                                                             │
│ Logic:                                                      │
│   CRITICAL: unencrypted storage in prod                    │
│   HIGH:     public access in any env                       │
│   MEDIUM:   missing tags, logging disabled                 │
│   LOW:      cost optimization                              │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼────────────────┐
         ▼               ▼                ▼
    CRITICAL           HIGH            MEDIUM/LOW
         │               │                │
Lớp 3: RESPOND (Phản Hồi)
         ▼               ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐
│ SSM Automation│ │ SNS → Human │ │ Lambda → Jira API    │
│ (auto-fix)   │ │ Approval     │ │ (create ticket)      │
│              │ │ (email/Slack)│ │                      │
│ Example:     │ │              │ │ Ticket: resource ID  │
│ Encrypt EBS  │ │ Approve?     │ │ rule violation       │
│ volume       │ │ Y → SSM fix  │ │ SLA: 48 hours        │
│              │ │ N → Jira     │ │                      │
└──────┬───────┘ └──────────────┘ └──────────────────────┘
       │
       ▼ (All paths)
┌─────────────────────────────────────────────────────────────┐
│ CloudTrail: ghi lại mọi remediation action                  │
│ DynamoDB: tracking table (rule, resource, action, timestamp)│
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
Lớp 4: REPORT (Báo Cáo)
┌─────────────────────────────────────────────────────────────┐
│ EventBridge Scheduler: mỗi thứ Sáu 17:00                   │
│   → Lambda: generate_compliance_report()                    │
│     → Config Aggregator API: get all findings               │
│     → DynamoDB: get remediation history                     │
│     → Generate HTML report                                  │
│     → SES: email to CISO, Security Team                     │
└─────────────────────────────────────────────────────────────┘
```

### Lambda Classifier Logic

```python
import boto3
import json

CRITICAL_RULES = [
    'rds-storage-encrypted',
    'ebs-encrypted-volumes',
    's3-bucket-server-side-encryption-enabled'
]

HIGH_RULES = [
    's3-bucket-public-read-prohibited',
    's3-bucket-public-write-prohibited',
    'restricted-ssh'
]

def classify_violation(config_rule_name, resource_type, environment):
    if environment == 'production' and config_rule_name in CRITICAL_RULES:
        return 'CRITICAL'
    elif config_rule_name in HIGH_RULES:
        return 'HIGH'
    elif config_rule_name in CRITICAL_RULES:
        return 'MEDIUM'
    else:
        return 'LOW'

def lambda_handler(event, context):
    detail = event['detail']
    config_rule = detail['configRuleName']
    resource_id = detail['resourceId']
    resource_type = detail['resourceType']
    
    # Lấy environment tag từ resource
    tags = get_resource_tags(resource_type, resource_id)
    environment = tags.get('Environment', 'unknown')
    
    severity = classify_violation(config_rule, resource_type, environment)
    
    if severity == 'CRITICAL':
        trigger_auto_remediation(config_rule, resource_id)
        send_alert('critical-alerts', config_rule, resource_id, severity)
    elif severity == 'HIGH':
        send_approval_request(config_rule, resource_id)
    else:
        create_jira_ticket(config_rule, resource_id, severity)
    
    log_to_dynamodb(config_rule, resource_id, severity, 'TRIGGERED')
```

### Trade-offs Cần Thảo Luận

| Quyết Định                          | Option A                        | Option B                          | Chọn & Lý Do                    |
| ----------------------------------- | ------------------------------- | --------------------------------- | -------------------------------- |
| Auto-remediation scope              | Chỉ dev/staging                 | Cả production với approval         | Production cần approval — safety first |
| Ticket system                       | Jira (tự tích hợp)             | AWS Systems Manager OpsCenter     | Jira — team đã quen              |
| Report format                       | HTML email                      | S3 + QuickSight dashboard          | QuickSight nếu cần trend overtime |
| Tracking store                      | DynamoDB                        | CloudWatch custom metrics          | DynamoDB — queryable, flexible    |
| Approval workflow                   | Email link                      | Slack bot với /approve command     | Slack — faster response           |

---

## Scenario S4: Enterprise Landing Zone Design

### Yêu Cầu

**Bối cảnh:** Financial institution (ngân hàng) với 200 developers, 50 applications, cần design AWS landing zone đáp ứng SOC2 Type II + PCI-DSS + GDPR.

**Requirements:**
- R1: Isolation hoàn toàn giữa cardholder data và các workload khác (PCI-DSS)
- R2: EU data phải ở trong EU region (GDPR data residency)
- R3: Zero-trust network model
- R4: Centralized egress với firewall inspection
- R5: GitOps-based account provisioning
- R6: SSO với existing Active Directory on-premises

---

### Kiến Trúc Đề Xuất (Cấp Cao)

```
On-Premises
├── Active Directory Domain Controller
└── AWS Direct Connect (10Gbps) ─────────────┐
                                              │
AWS Organization (Management Account)        │
├── OU: Security (eu-west-1 primary)         │
│   ├── Log Archive Account                  │
│   │   └── S3: audit logs (Lifecycle → Glacier 90d)
│   └── Security Tooling Account             │
│       ├── Security Hub (aggregator)        │
│       ├── GuardDuty (master)               │
│       └── Macie (S3 data classification)   │
│                                            │
├── OU: Network Hub                          │
│   └── Network Account (per region)         │
│       ├── Transit Gateway                  │
│       ├── AWS Network Firewall (centralized egress)
│       ├── VPC Inspection (East-West traffic)
│       └── NAT Gateway (centralized)        │
│                                            │
├── OU: Identity                             │
│   └── Identity Account                    │
│       ├── IAM Identity Center (SSO)        │
│       │   └── AD Connector ←──────────────┘
│       └── AWS Directory Service            │
│                                            │
├── OU: PCI-DSS Scope (Isolated)            │
│   ├── PCI-App Account (us-east-1)         │
│   └── PCI-Data Account (us-east-1)        │
│   SCPs: Deny non-us-east-1 resources      │
│         Deny unencrypted storage          │
│         Deny public internet access       │
│                                            │
├── OU: GDPR Workloads (EU only)            │
│   ├── EU-App Account (eu-west-1)          │
│   └── EU-Data Account (eu-central-1)     │
│   SCPs: Deny non-EU regions              │
│         Deny cross-region data transfer   │
│                                            │
└── OU: Standard Workloads                  │
    ├── Team-A-Prod
    ├── Team-A-Dev
    └── ... (Account Factory provisioned)
```

### GitOps Account Provisioning

```yaml
# Git Repository: aws-accounts/
# Mỗi file = 1 account request
# PR → Review → Merge → CI/CD → Account created

# file: accounts/payments-team-prod.yaml
apiVersion: v1
kind: AWSAccount
metadata:
  name: payments-team-prod
spec:
  email: aws+payments-prod@bank.com
  organizationalUnit: OU-Standard-Production
  owner:
    team: payments
    lead: john.doe@bank.com
  compliance:
    pciDss: false
    gdpr: false
    sox: true
  network:
    vpcCidr: 10.50.0.0/16
    connectToTransitGateway: true
  budget:
    monthly: 15000
    alertThreshold: 80
  ssoGroups:
    - payments-developers (ReadOnly)
    - payments-leads (PowerUser)
    - devops-team (Admin)
```

```python
# CI/CD Pipeline (GitHub Actions + Python)
def provision_account(account_spec):
    # 1. Tạo account qua Organizations API
    account_id = create_account(account_spec)
    
    # 2. Move vào đúng OU
    move_to_ou(account_id, account_spec['ou'])
    
    # 3. Apply baseline CloudFormation StackSet
    deploy_baseline(account_id, account_spec)
    
    # 4. Setup SSO assignments
    configure_sso(account_id, account_spec['ssoGroups'])
    
    # 5. Create Jira ticket confirm
    create_jira_ticket(account_id, account_spec)
```

---

## Scenario S5: Incident Response Automation

### Yêu Cầu

**Bối cảnh:** E-commerce platform 10M users, yêu cầu MTTR (Mean Time To Recovery — Thời Gian Trung Bình Phục Hồi) < 15 phút cho P1 incidents.

**Requirements:**
- R1: Auto-detect và escalate P1 incidents trong < 3 phút
- R2: Auto-scale khi phát hiện traffic spike trước khi latency tăng
- R3: Auto-rollback deployment nếu error rate tăng sau deploy
- R4: Runbook tự động thu thập diagnostic info khi incident
- R5: Post-incident report tự động

---

### Kiến Trúc Đề Xuất

```
Detection Layer:
┌────────────────────────────────────────────────────────────┐
│ CloudWatch Metrics → Composite Alarm                       │
│                                                            │
│ P1 Alarm = (5xx > 100/phút)                              │
│            AND (P99 Latency > 5s)                         │
│            AND (HealthyHost < MinCapacity)                │
└────────────────────────┬───────────────────────────────────┘
                         │ ALARM state
                         ▼
Orchestration Layer:
┌────────────────────────────────────────────────────────────┐
│ EventBridge → Step Functions State Machine                 │
│                                                            │
│ Workflow:                                                  │
│ 1. Notify On-Call (PagerDuty P1)                          │
│ 2. Collect Diagnostics (parallel):                        │
│    2a. CloudWatch Logs Insights — recent errors           │
│    2b. CloudTrail — recent API changes                    │
│    2c. Config Timeline — resource changes                 │
│    2d. X-Ray — distributed traces                        │
│ 3. Check: recent deployment? (CodeDeploy last 30 min)     │
│    Yes → Auto-rollback                                    │
│    No → Try auto-remediation (scaling)                    │
│ 4. Re-evaluate: incident resolved?                        │
│    Yes → Close, generate report                          │
│    No → Escalate to CTO, page backup on-call            │
│ 5. Generate post-incident report (S3 + Email)            │
└────────────────────────────────────────────────────────────┘

Auto-Remediation Runbooks (SSM Automation):
┌─────────────────────────────────────────────────────────────┐
│ Runbook 1: Scale Out                                        │
│   → ECS Service: desired count × 2                         │
│   → Wait 3 minutes                                         │
│   → Verify: HealthyHost > MinCapacity                      │
│                                                             │
│ Runbook 2: Rollback Deployment                              │
│   → CodeDeploy: get previous successful deployment         │
│   → Trigger rollback                                       │
│   → Wait until deployment complete                         │
│   → Verify: error rate < 0.1%                             │
│                                                             │
│ Runbook 3: Database Failover                                │
│   → RDS: trigger failover to read replica                  │
│   → Update Parameter Store with new endpoint               │
│   → Trigger ECS service restart                            │
└─────────────────────────────────────────────────────────────┘
```

### Step Functions Auto-Rollback

```json
{
  "Comment": "Incident Response Automation",
  "StartAt": "NotifyOnCall",
  "States": {
    "NotifyOnCall": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123:pagerduty-p1",
        "Message.$": "States.Format('P1 INCIDENT: {} - {}', $.alarmName, $.timestamp)"
      },
      "Next": "CollectDiagnosticsParallel"
    },
    "CollectDiagnosticsParallel": {
      "Type": "Parallel",
      "Branches": [
        {"StartAt": "QueryCloudWatchLogs", "States": {}},
        {"StartAt": "QueryCloudTrail", "States": {}},
        {"StartAt": "GetRecentDeployments", "States": {}}
      ],
      "Next": "CheckRecentDeployment"
    },
    "CheckRecentDeployment": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.recentDeployment",
        "BooleanEquals": true,
        "Next": "AutoRollback"
      }],
      "Default": "AutoScale"
    },
    "AutoRollback": {
      "Type": "Task",
      "Resource": "arn:aws:states:::codedeploy:rollback",
      "Next": "WaitAndVerify"
    }
  }
}
```

---

## Scenario S6: FinOps Governance Cho Multi-Account (100+ Accounts)

### Yêu Cầu

**Bối cảnh:** Tập đoàn công nghệ với 50 teams, 120 AWS accounts, bill hàng tháng $2M+. CFO muốn: chargeback chính xác theo team, phát hiện waste, và dự báo chi phí.

**Requirements:**
- R1: Chargeback (quy về chi phí từng team) chính xác ± 5%
- R2: Alert khi team vượt 90% budget trong vòng 24 giờ
- R3: Phát hiện unused resources tự động hàng ngày
- R4: Savings Plans optimization hàng tháng
- R5: Cost forecast 3 tháng tới với độ chính xác ± 10%

---

### Kiến Trúc Đề Xuất

```
Data Collection Layer:
┌─────────────────────────────────────────────────────────────┐
│ AWS Cost and Usage Report (CUR)                             │
│ → S3 Bucket (Cost Management Account)                      │
│ → Athena Table (partitioned by year/month/account)         │
│ → QuickSight Dashboard                                     │
│                                                             │
│ Tag Governance:                                             │
│ Organizations → Tag Policy:                                 │
│   Required tags: Team, Project, CostCenter, Environment    │
│   Enforced via: SCP + AWS Config Rule                      │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ Chargeback Engine (Lambda + Athena)                         │
│                                                             │
│ Monthly Job (1st of month):                                 │
│ 1. Query CUR: cost per team tag                            │
│ 2. Shared cost allocation:                                  │
│    - Transit Gateway: allocate by data transfer            │
│    - Shared services: allocate by account count            │
│    - Support costs: allocate by % of total spend           │
│ 3. Generate per-team invoice (PDF + CSV)                   │
│ 4. Send to finance system (SAP/NetSuite via API)           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ Budget Monitoring (per team = per account)                  │
│                                                             │
│ AWS Budgets per account:                                    │
│ - Alert 80%: Slack notification to team lead               │
│ - Alert 100%: PagerDuty + freeze non-prod resources         │
│ - Action 100%: Deny policy on IAM for new resources        │
│                                                             │
│ Cost Anomaly Detection:                                     │
│ - Monitor per service per account                          │
│ - Threshold: > 20% increase week-over-week                 │
│ - Alert: Slack → #finops channel                           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ Waste Detection (Daily Lambda Job)                          │
│                                                             │
│ Trusted Advisor API:                                        │
│ - Low utilization EC2 (CPU < 10%, 14 days)                │
│ - Unassociated Elastic IPs                                  │
│ - Idle Load Balancers                                       │
│ - Underutilized RDS                                        │
│                                                             │
│ Custom Checks:                                              │
│ - EBS volumes not attached (> 7 days)                      │
│ - ECS services with 0 desired tasks                        │
│ - Lambda functions not invoked (> 30 days)                 │
│                                                             │
│ Output: Waste Report → Jira tickets → team leads           │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│ Savings Plans Optimization (Monthly)                        │
│                                                             │
│ Cost Explorer API:                                          │
│ - GetSavingsPlansPurchaseRecommendation                    │
│ - Coverage analysis: on-demand vs SP-covered               │
│ - Target: 80% covered by Savings Plans                     │
│                                                             │
│ Auto-recommendation report → FinOps team review            │
│ → Purchase via API (with CFO approval workflow)            │
└─────────────────────────────────────────────────────────────┘
```

### Athena Query — Chargeback Report

```sql
-- Monthly chargeback per team
WITH tagged_costs AS (
  SELECT
    line_item_usage_account_id AS account_id,
    resource_tags_user_team AS team,
    resource_tags_user_project AS project,
    SUM(line_item_blended_cost) AS blended_cost,
    SUM(line_item_unblended_cost) AS unblended_cost
  FROM cur_table
  WHERE year = '2026' AND month = '05'
    AND resource_tags_user_team IS NOT NULL
  GROUP BY 1, 2, 3
),
untagged_costs AS (
  SELECT
    line_item_usage_account_id AS account_id,
    'UNTAGGED' AS team,
    'UNTAGGED' AS project,
    SUM(line_item_blended_cost) AS blended_cost,
    SUM(line_item_unblended_cost) AS unblended_cost
  FROM cur_table
  WHERE year = '2026' AND month = '05'
    AND resource_tags_user_team IS NULL
  GROUP BY 1
)
SELECT * FROM tagged_costs
UNION ALL
SELECT * FROM untagged_costs
ORDER BY blended_cost DESC;
```

### Trade-offs Cần Thảo Luận

| Quyết Định                     | Option A                         | Option B                          | Chọn & Lý Do                     |
| ------------------------------ | -------------------------------- | --------------------------------- | --------------------------------- |
| BI tool                        | QuickSight (AWS native)          | Looker / Tableau                  | QuickSight nếu team chưa có BI    |
| Chargeback model               | 100% per tag                     | 80% per tag + 20% shared pool     | Shared pool thực tế hơn           |
| Waste detection                | Trusted Advisor API              | Custom CloudWatch metrics         | TA API đủ dùng, ít code hơn       |
| Budget enforcement             | IAM deny via Budget Action       | SCP per account                   | Budget Action vì linh hoạt hơn    |
| Forecast engine                | Cost Explorer API                | Custom ML model                   | CES đủ cho 3 tháng, không over-engineer |

---

## Checklist Đánh Giá System Design

Sau khi thiết kế xong, tự kiểm tra:

### Reliability (Độ Tin Cậy)
- [ ] Có single point of failure nào không?
- [ ] Nếu một service down, hệ thống còn hoạt động không?
- [ ] Data được backup không? Recovery plan là gì?

### Security (Bảo Mật)
- [ ] Least privilege cho tất cả IAM roles?
- [ ] Encryption at rest và in transit?
- [ ] Network isolation đủ chưa?
- [ ] Audit trail đầy đủ?

### Scalability (Khả Năng Mở Rộng)
- [ ] Khi số accounts tăng 10x, kiến trúc còn hoạt động không?
- [ ] Lambda có thể bị throttle không? Cần concurrency limit?

### Cost (Chi Phí)
- [ ] Ước tính chi phí hàng tháng?
- [ ] Có tài nguyên nào dư thừa không cần thiết?
- [ ] Config Rule tính phí theo số config items — có optimize không?

### Operational Excellence (Xuất Sắc Vận Hành)
- [ ] Runbook cho sự cố phổ biến đã có chưa?
- [ ] On-call rotation được cấu hình chưa?
- [ ] Monitoring cho chính monitoring system chưa?

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
