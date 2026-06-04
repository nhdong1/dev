# SSM Run Command & Automation — Thực Thi Lệnh & Tự Động Hóa Vận Hành

> **Run Command** cho phép thực thi lệnh shell hoặc PowerShell trên fleet EC2 mà không cần SSH. **Automation** (Tự Động Hóa) mở rộng khả năng này thành runbook (sổ tay vận hành tự động) nhiều bước, tích hợp với toàn bộ hệ sinh thái AWS.

---

## 📚 Mục Lục

1. [Run Command — Thực Thi Lệnh Hàng Loạt](#run-command--thực-thi-lệnh-hàng-loạt)
2. [SSM Documents — Tài Liệu Lệnh](#ssm-documents--tài-liệu-lệnh)
3. [SSM Automation — Runbook Tự Động](#ssm-automation--runbook-tự-động)
4. [So Sánh Run Command vs Automation](#so-sánh-run-command-vs-automation)
5. [Approval Gates & Human Approval](#approval-gates--human-approval)
6. [Tình Huống Thực Tế](#tình-huống-thực-tế)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Run Command — Thực Thi Lệnh Hàng Loạt

### Run Command Là Gì?

**Run Command** (Chạy Lệnh) thực thi lệnh một bước trên một hoặc nhiều managed instances cùng lúc — sử dụng SSM Agent, không cần SSH, không cần network connectivity.

### Kiến Trúc Run Command

```
Người Dùng / EventBridge / Lambda
         │
         ▼ API: ssm:SendCommand
    SSM Service
         │
         │ WebSocket channel (qua SSM Agent)
         ├──────────────────────────────────▶ EC2-instance-1
         ├──────────────────────────────────▶ EC2-instance-2
         ├──────────────────────────────────▶ EC2-instance-3
         │                                      ...
         ▼
    Kết quả (output) → CloudWatch Logs / S3
```

### Ví Dụ Run Command Cơ Bản

```bash
# Chạy lệnh shell trên tất cả EC2 có tag Environment=Production
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"commands":["df -h","free -m","systemctl status nginx"]}' \
  --comment "Health check on production fleet" \
  --output-s3-bucket-name "my-ssm-output-bucket" \
  --output-s3-key-prefix "run-command/health-check" \
  --cloud-watch-output-config '{
    "CloudWatchOutputEnabled": true,
    "CloudWatchLogGroupName": "/ssm/run-command"
  }'

# Lấy Command ID từ output, theo dõi tiến trình
COMMAND_ID="abc12345-1234-1234-1234-abc1234567890"

aws ssm list-command-invocations \
  --command-id $COMMAND_ID \
  --details \
  --query 'CommandInvocations[*].[InstanceId,Status,StatusDetails]' \
  --output table
```

### Kiểm Soát Concurrency (Song Song Hóa)

```bash
# Chạy tuần tự — chỉ 1 instance cùng lúc (an toàn cho production)
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"commands":["sudo systemctl restart myapp"]}' \
  --max-concurrency "1" \
  --max-errors "0"

# Chạy song song 10% fleet (rolling restart)
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Patch Group,Values=web-servers" \
  --parameters '{"commands":["sudo yum update -y --security"]}' \
  --max-concurrency "10%" \
  --max-errors "5%"
```

| Tham Số | Ý Nghĩa |
|---------|---------|
| `max-concurrency` | Số/% instances chạy cùng lúc |
| `max-errors` | Số/% lỗi cho phép trước khi dừng |
| `--comment` | Ghi chú hiển thị trong audit |
| `--timeout-seconds` | Timeout mỗi invocation (tối đa 172800s = 48h) |

### Các Document AWS-Managed Phổ Biến

| Document | Mục Đích |
|----------|---------|
| `AWS-RunShellScript` | Chạy shell script trên Linux |
| `AWS-RunPowerShellScript` | Chạy PowerShell trên Windows |
| `AWS-RunPatchBaseline` | Quét/vá lỗi theo Patch Baseline |
| `AWS-UpdateSSMAgent` | Cập nhật SSM Agent |
| `AWS-ConfigureAWSPackage` | Cài/xóa packages (Distributor) |
| `AWS-RunAnsiblePlaybook` | Chạy Ansible playbook |
| `AWS-ApplyChefRecipes` | Chạy Chef recipes |

---

## SSM Documents — Tài Liệu Lệnh

### SSM Document Là Gì?

**SSM Document** (Tài Liệu SSM) là file JSON hoặc YAML định nghĩa các hành động (actions) mà SSM thực hiện. Có thể ví như "template" cho lệnh hoặc automation.

### Các Loại Document

| Loại | Dùng Cho | Ví Dụ |
|------|---------|-------|
| **Command** | Run Command | Chạy script, install software |
| **Automation** | SSM Automation | Multi-step runbooks |
| **Session** | Session Manager | Cấu hình session preferences |
| **Package** | Distributor | Đóng gói phần mềm |
| **Policy** | State Manager | Cấu hình compliance |

### Tạo Custom Command Document

```yaml
# File: my-app-health-check.yml
schemaVersion: "2.2"
description: "Health check cho MyApp"
parameters:
  AppPort:
    type: String
    default: "8080"
    description: "Application port"
  Endpoint:
    type: String
    default: "/health"
    description: "Health check endpoint"
mainSteps:
  - action: aws:runShellScript
    name: checkDiskSpace
    inputs:
      runCommand:
        - |
          DISK_USAGE=$(df / | awk 'NR==2{print $5}' | tr -d '%')
          if [ "$DISK_USAGE" -gt 85 ]; then
            echo "WARN: Disk usage is ${DISK_USAGE}%"
          else
            echo "OK: Disk usage is ${DISK_USAGE}%"
          fi
  - action: aws:runShellScript
    name: checkAppHealth
    inputs:
      runCommand:
        - |
          RESPONSE=$(curl -sf -o /dev/null -w "%{http_code}" \
            http://localhost:{{ AppPort }}{{ Endpoint }})
          if [ "$RESPONSE" = "200" ]; then
            echo "OK: App responding on port {{ AppPort }}"
          else
            echo "FAIL: App returned HTTP $RESPONSE"
            exit 1
          fi
  - action: aws:runShellScript
    name: checkMemory
    inputs:
      runCommand:
        - free -h
        - ps aux --sort=-%mem | head -10
```

```bash
# Tạo document từ file
aws ssm create-document \
  --name "MyApp-HealthCheck" \
  --document-type Command \
  --document-format YAML \
  --content file://my-app-health-check.yml

# Sử dụng
aws ssm send-command \
  --document-name "MyApp-HealthCheck" \
  --targets "Key=tag:App,Values=myapp" \
  --parameters '{"AppPort":["3000"],"Endpoint":["/api/health"]}'
```

---

## SSM Automation — Runbook Tự Động

### Automation Là Gì?

**SSM Automation** thực thi **runbook** (sổ tay vận hành tự động) — workflow nhiều bước phức tạp, có thể tạm dừng để chờ phê duyệt của con người, gọi các AWS API, và phối hợp nhiều dịch vụ.

### So Sánh Run Command vs Automation

```
Run Command:
  User → [Send Command] → EC2 Instances
         (Single step, immediate execution)

Automation:
  User → [Start Automation] → Step 1 → Step 2 → [Pause for Approval]
         → Step 3 → AWS API → Step 4 → EC2 → Step 5 → Done
         (Multi-step, stateful, can pause and resume)
```

### Các Action Types Trong Automation

| Action | Mô Tả |
|--------|-------|
| `aws:runCommand` | Gọi SSM Run Command |
| `aws:executeAwsApi` | Gọi bất kỳ AWS API nào |
| `aws:waitForAwsResourceProperty` | Chờ resource đạt trạng thái mong muốn |
| `aws:approve` | Dừng và chờ phê duyệt thủ công |
| `aws:createImage` | Tạo EC2 AMI |
| `aws:changeInstanceState` | Start/Stop/Terminate EC2 |
| `aws:branch` | Rẽ nhánh dựa trên điều kiện |
| `aws:sleep` | Chờ một khoảng thời gian |
| `aws:invokeLambdaFunction` | Gọi Lambda function |
| `aws:executeScript` | Chạy Python hoặc PowerShell |

### Ví Dụ: Automation Tạo Patched AMI

```yaml
schemaVersion: "0.3"
description: "Tạo AMI đã được patch từ instance hiện tại"
parameters:
  SourceInstanceId:
    type: String
    description: "Instance ID để tạo AMI từ đó"
  NotificationTopicArn:
    type: String
    description: "SNS topic để thông báo kết quả"
assumeRole: "arn:aws:iam::123456789:role/SSMAutomationRole"
mainSteps:
  - name: stopInstance
    action: aws:changeInstanceState
    inputs:
      InstanceIds:
        - "{{ SourceInstanceId }}"
      DesiredState: stopped

  - name: waitForStop
    action: aws:waitForAwsResourceProperty
    inputs:
      Service: ec2
      Api: DescribeInstances
      InstanceIds:
        - "{{ SourceInstanceId }}"
      PropertySelector: "$.Reservations[0].Instances[0].State.Name"
      DesiredValues:
        - stopped

  - name: createAMI
    action: aws:createImage
    inputs:
      InstanceId: "{{ SourceInstanceId }}"
      ImageName: "PatchedAMI-{{ SourceInstanceId }}-{{ global:DATE_TIME }}"
      NoReboot: true

  - name: startInstance
    action: aws:changeInstanceState
    inputs:
      InstanceIds:
        - "{{ SourceInstanceId }}"
      DesiredState: running

  - name: notifySuccess
    action: aws:executeAwsApi
    inputs:
      Service: sns
      Api: Publish
      TopicArn: "{{ NotificationTopicArn }}"
      Message: "AMI created successfully: {{ createAMI.ImageId }}"
      Subject: "Patched AMI Ready"
```

```bash
# Chạy automation
aws ssm start-automation-execution \
  --document-name "CreatePatchedAMI" \
  --parameters '{
    "SourceInstanceId": ["i-1234567890abcdef0"],
    "NotificationTopicArn": ["arn:aws:sns:ap-southeast-1:123456789:ops-team"]
  }'

# Theo dõi trạng thái
aws ssm describe-automation-executions \
  --filters "Key=ExecutionStatus,Values=InProgress" \
  --query 'AutomationExecutionMetadataList[*].[AutomationExecutionId,DocumentName,ExecutionStatus]'
```

### Automation Với Rate Control

```bash
# Chạy automation trên nhiều instances (Concurrency Control)
aws ssm start-automation-execution \
  --document-name "AWS-RestartEC2Instance" \
  --targets "Key=tag:Environment,Values=Production" \
  --target-parameter-name "InstanceId" \
  --max-concurrency "2" \
  --max-errors "1"
```

---

## Approval Gates & Human Approval

### Tại Sao Cần Human Approval?

Trong production, một số thao tác nguy hiểm cần người có thẩm quyền xem xét và phê duyệt trước khi thực thi — ví dụ: terminate database instance, deploy release mới, rollback production.

### Cách Hoạt Động

```
Automation chạy đến step "aws:approve"
         │
         ▼
Gửi notification đến SNS topic
         │
         ▼
Ops engineer nhận email/Slack
         │
         │ Phê duyệt (Approve) → Automation tiếp tục
         │ Từ chối (Reject)    → Automation dừng + ghi log
         │
         ▼
Automation hoàn thành (hoặc thất bại)
```

### Ví Dụ Document Có Approval Gate

```yaml
schemaVersion: "0.3"
description: "Deploy phiên bản mới với approval gate"
parameters:
  NewVersion:
    type: String
  ApproverArn:
    type: String
    description: "IAM ARN của người phê duyệt"

mainSteps:
  - name: prepareDeployment
    action: aws:runCommand
    inputs:
      DocumentName: AWS-RunShellScript
      Targets:
        - Key: "tag:Environment"
          Values: ["Production"]
      Parameters:
        commands:
          - echo "Preparing deployment of {{ NewVersion }}"

  - name: approveDeployment
    action: aws:approve
    timeoutSeconds: 86400  # 24 giờ để phê duyệt
    inputs:
      NotificationArn: "arn:aws:sns:ap-southeast-1:123456789:ops-approvals"
      Message: "Deployment of version {{ NewVersion }} requires approval. Review changes at: https://changelog.internal/{{ NewVersion }}"
      MinRequiredApprovals: 2
      Approvers:
        - "{{ ApproverArn }}"
        - "arn:aws:iam::123456789:role/TechLeadRole"

  - name: executeDeployment
    action: aws:runCommand
    inputs:
      DocumentName: AWS-RunShellScript
      Targets:
        - Key: "tag:Environment"
          Values: ["Production"]
      Parameters:
        commands:
          - /opt/deploy/deploy.sh {{ NewVersion }}
```

---

## Tình Huống Thực Tế

### Tình Huống 1: Incident Response — Thu Thập Logs Nhanh

```bash
# Khi có sự cố: thu thập diagnostic info từ tất cả production servers
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "commands": [
      "echo === SYSTEM INFO ===",
      "uname -a && uptime",
      "echo === DISK USAGE ===",
      "df -h",
      "echo === MEMORY ===",
      "free -m",
      "echo === TOP PROCESSES ===",
      "ps aux --sort=-%cpu | head -20",
      "echo === RECENT ERRORS ===",
      "journalctl -p err -n 50 --no-pager",
      "echo === APP LOGS ===",
      "tail -100 /var/log/myapp/error.log 2>/dev/null || echo No app logs"
    ]
  }' \
  --output-s3-bucket-name "incident-diagnostics-$(date +%Y%m%d)" \
  --output-s3-key-prefix "incident-$(date +%H%M%S)/" \
  --comment "Incident response: collecting diagnostics"
```

### Tình Huống 2: Automated Remediation Với EventBridge

```
Config Rule phát hiện vi phạm
         │
         ▼
EventBridge Rule bắt sự kiện
         │
         ▼
Lambda trigger SSM Automation
         │
         ▼
Automation tự sửa vi phạm
         │
         ▼
Gửi notification kết quả
```

```python
# Lambda trigger SSM Automation khi Config phát hiện S3 public bucket
import boto3

ssm = boto3.client('ssm')

def lambda_handler(event, context):
    detail = event['detail']
    resource_id = detail['resourceId']  # S3 bucket name
    
    response = ssm.start_automation_execution(
        DocumentName='AWS-DisableS3BucketPublicReadWrite',
        Parameters={
            'BucketName': [resource_id],
            'AutomationAssumeRole': ['arn:aws:iam::123456789:role/AutoRemediationRole']
        }
    )
    
    return {
        'executionId': response['AutomationExecutionId']
    }
```

### Tình Huống 3: Scheduled Maintenance Automation

```bash
# Tạo CloudWatch Events rule để chạy automation mỗi Chủ nhật
aws events put-rule \
  --name "WeeklyMaintenanceAutomation" \
  --schedule-expression "cron(0 2 ? * SUN *)" \
  --state ENABLED

# Target là SSM Automation
aws events put-targets \
  --rule "WeeklyMaintenanceAutomation" \
  --targets '[{
    "Id": "SSMAutomation",
    "Arn": "arn:aws:ssm:ap-southeast-1:123456789:automation-definition/WeeklyMaintenance",
    "RoleArn": "arn:aws:iam::123456789:role/EventsToSSMRole",
    "Input": "{\"Environment\":[\"Production\"]}"
  }]'
```

---

## So Sánh Run Command vs Automation

| Tiêu Chí | Run Command | Automation |
|----------|-------------|-----------|
| **Số bước** | Một bước | Nhiều bước (workflow) |
| **Targets** | Chỉ EC2/managed instances | EC2 + AWS API + Lambda + RDS... |
| **Approval** | ❌ Không có | ✅ Có (aws:approve) |
| **Rollback** | ❌ Không | ✅ Có (điều kiện rẽ nhánh) |
| **Timeout** | Instance level | Workflow level + step level |
| **Trigger** | Manual, EventBridge, Lambda | Manual, Maintenance Window, EventBridge |
| **Output** | S3, CloudWatch Logs | S3, CloudWatch Logs, return variables |
| **Use case** | Shell commands, patches | Deploy, AMI creation, incident response |

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Run Command khác Automation như thế nào? Khi nào dùng cái nào?**

> Run Command: Thực thi lệnh đơn giản (một bước) trực tiếp trên fleet instances — phù hợp cho patch, restart service, chạy health check. Automation: Workflow nhiều bước phức tạp, có thể gọi AWS API, tạo AMI, chờ phê duyệt của con người — phù hợp cho deploy pipeline, incident response, create/update infrastructure.

**Q: max-concurrency và max-errors trong Run Command có ý nghĩa gì?**

> `max-concurrency` giới hạn số instances chạy lệnh cùng một lúc (số hoặc %). Ví dụ `"10%"` trên 100 instances = chạy 10 cùng lúc, chờ xong rồi tiếp 10 instances tiếp theo — giúp tránh ảnh hưởng toàn bộ fleet. `max-errors` dừng toàn bộ command nếu số lỗi vượt ngưỡng — ví dụ `"5%"` trên 100 instances = dừng nếu có hơn 5 instances lỗi.

**Q: SSM Document có thể chia sẻ giữa các AWS account không?**

> Có. Dùng `modify-document-permission` để share document với account cụ thể hoặc public. Thường dùng khi công ty muốn chuẩn hóa runbooks cho nhiều account trong Organizations.

### Nâng Cao

**Q: Thiết kế automated incident response pipeline khi CloudWatch Alarm báo CPU > 90%?**

> **(1)** CloudWatch Alarm (CPU > 90%) → trigger SNS topic.
>
> **(2)** SNS → EventBridge rule → Lambda function.
>
> **(3)** Lambda phân tích alarm context, xác định instance bị ảnh hưởng, gọi `ssm:StartAutomationExecution`.
>
> **(4)** Automation Document chạy các bước:
> - Step 1: `aws:runCommand` — Thu thập `ps aux`, `netstat`, application logs.
> - Step 2: `aws:executeScript` — Phân tích logs, tìm potential root cause.
> - Step 3: `aws:approve` — Gửi findings đến ops team qua SNS/Slack, chờ approval để thực hiện action.
> - Step 4 (nếu approved): `aws:executeAwsApi` — Tăng scale ASG, hoặc restart process.
> - Step 5: `aws:runCommand` — Health check sau khi action.
> - Step 6: `aws:executeAwsApi` — Publish kết quả vào CloudWatch custom metric.
>
> **(5)** Toàn bộ log lưu vào S3 cho post-mortem.

---

**Liên Quan:** [Session Manager](./1-session-manager.md) | [Patch Manager](./2-patch-manager.md) | [Inventory & Compliance](./5-inventory-compliance.md)
