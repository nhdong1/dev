# SSM Distributor & OpsCenter — Phân Phối Phần Mềm & Quản Lý Sự Cố Vận Hành

> **SSM Distributor** (Trình Phân Phối SSM) đóng gói và phân phối phần mềm tùy chỉnh hoặc AWS agents đến managed instances. **OpsCenter** (Trung Tâm Vận Hành) tập hợp và quản lý các vấn đề vận hành (operational issues) từ nhiều nguồn vào một nơi duy nhất để điều tra và xử lý.

---

## 📚 Mục Lục

1. [SSM Distributor — Phân Phối Phần Mềm](#ssm-distributor--phân-phối-phần-mềm)
2. [Tạo Package Distributor](#tạo-package-distributor)
3. [Cài Đặt Packages Từ Distributor](#cài-đặt-packages-từ-distributor)
4. [AWS-Managed Packages](#aws-managed-packages)
5. [OpsCenter — Trung Tâm Vận Hành](#opscenter--trung-tâm-vận-hành)
6. [OpsItems — Mục Vận Hành](#opsitems--mục-vận-hành)
7. [Tích Hợp OpsCenter Với AWS Services](#tích-hợp-opscenter-với-aws-services)
8. [Runbook Tự Động Từ OpsCenter](#runbook-tự-động-từ-opscenter)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## SSM Distributor — Phân Phối Phần Mềm

### Distributor Là Gì?

**SSM Distributor** (Distributor — Trình Phân Phối) cho phép đóng gói (package) phần mềm của bạn thành định dạng SSM package, sau đó phân phối đến hàng trăm managed instances chỉ bằng một lệnh — thay vì copy thủ công hoặc dùng Ansible/Chef chỉ cho việc này.

### Vấn Đề Distributor Giải Quyết

```
Trước khi có Distributor:
┌──────────────────────────────────────────────────────────┐
│  Cài custom agent trên 200 EC2:                          │
│  1. Build binary                                         │
│  2. Upload lên S3                                        │
│  3. SSH vào từng machine (hoặc Ansible)                  │
│  4. Download từ S3                                       │
│  5. Unzip, configure, install                            │
│  6. Verify installation                                  │
│  7. Handle different OS versions (Amazon Linux 2,        │
│     Amazon Linux 2023, Ubuntu 20.04, 22.04, Windows...)  │
│                                                          │
│  → Rất phức tạp, dễ sai, khó rollback                   │
└──────────────────────────────────────────────────────────┘

Với Distributor:
┌──────────────────────────────────────────────────────────┐
│  1. Đóng gói thành SSM Package (một lần)                 │
│  2. aws ssm send-command                                 │
│     --document-name AWS-ConfigureAWSPackage             │
│     --parameters '{"action":"Install",                   │
│                    "name":"MyCustomAgent"}'              │
│  → Done! Tự động xử lý multi-OS, rollback, versioning    │
└──────────────────────────────────────────────────────────┘
```

### Cấu Trúc SSM Package

```
SSM Package (file .zip):
├── manifest.json          ← Metadata: tên, version, OS support
├── amazon_linux_2/
│   ├── install.sh         ← Script cài đặt cho Amazon Linux 2
│   └── uninstall.sh       ← Script gỡ cài đặt
├── amazon_linux_2023/
│   ├── install.sh
│   └── uninstall.sh
├── ubuntu_20.04/
│   ├── install.sh
│   └── uninstall.sh
└── windows/
    ├── install.ps1        ← PowerShell cho Windows
    └── uninstall.ps1
```

---

## Tạo Package Distributor

### manifest.json

```json
{
  "schemaVersion": "2.0",
  "version": "1.2.3",
  "publisher": "MyCompany",
  "packages": {
    "amazon_linux_2": {
      "x86_64": {
        "file": "amazon_linux_2.zip"
      },
      "arm64": {
        "file": "amazon_linux_2_arm64.zip"
      }
    },
    "amazon_linux_2023": {
      "x86_64": {
        "file": "amazon_linux_2023.zip"
      }
    },
    "ubuntu_20.04": {
      "x86_64": {
        "file": "ubuntu_2004.zip"
      }
    },
    "windows": {
      "x86_64": {
        "file": "windows.zip"
      }
    }
  },
  "files": {
    "amazon_linux_2.zip": {
      "checksums": {
        "sha256": "abc123..."
      }
    }
  }
}
```

### install.sh (Ví Dụ)

```bash
#!/bin/bash
set -e

# Lấy thông tin phiên bản từ manifest
VERSION="1.2.3"
INSTALL_DIR="/opt/mycompany/myagent"
SERVICE_NAME="myagent"

# Tạo thư mục
mkdir -p $INSTALL_DIR

# Copy binary
cp myagent-linux-x86_64 $INSTALL_DIR/myagent
chmod +x $INSTALL_DIR/myagent

# Lấy config từ SSM Parameter Store
DB_HOST=$(aws ssm get-parameter \
  --name "/prod/myagent/db/host" \
  --query 'Parameter.Value' \
  --output text)

# Tạo config file
cat > $INSTALL_DIR/config.yml << EOF
version: $VERSION
db_host: $DB_HOST
log_level: info
EOF

# Tạo systemd service
cat > /etc/systemd/system/$SERVICE_NAME.service << EOF
[Unit]
Description=MyCompany Agent
After=network.target

[Service]
Type=simple
ExecStart=$INSTALL_DIR/myagent
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable $SERVICE_NAME
systemctl start $SERVICE_NAME

echo "MyAgent $VERSION installed successfully"
```

### Upload Package Lên Distributor

```bash
# Bước 1: Upload files lên S3
aws s3 cp mypackage.zip s3://my-distributor-bucket/packages/myagent/1.2.3/

# Bước 2: Tạo package trong SSM Distributor
aws ssm create-document \
  --name "MyCompanyAgent" \
  --document-type Package \
  --attachments '{
    "Key": "SourceUrl",
    "Values": ["s3://my-distributor-bucket/packages/myagent/1.2.3/mypackage.zip"]
  }' \
  --version-name "1.2.3"

# Cập nhật phiên bản mới
aws ssm update-document \
  --name "MyCompanyAgent" \
  --document-version "$LATEST" \
  --version-name "1.3.0" \
  --attachments '{
    "Key": "SourceUrl",
    "Values": ["s3://my-distributor-bucket/packages/myagent/1.3.0/mypackage.zip"]
  }'
```

---

## Cài Đặt Packages Từ Distributor

### Cài Thủ Công Qua Run Command

```bash
# Cài phiên bản mới nhất
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["Install"],
    "name": ["MyCompanyAgent"],
    "version": [""]
  }' \
  --comment "Install MyCompanyAgent latest"

# Cài phiên bản cụ thể
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["Install"],
    "name": ["MyCompanyAgent"],
    "version": ["1.2.3"]
  }'

# Gỡ cài đặt
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["Uninstall"],
    "name": ["MyCompanyAgent"]
  }'
```

### Cài Tự Động Qua State Manager

```bash
# Đảm bảo MyCompanyAgent luôn được cài (State Manager)
aws ssm create-association \
  --name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:InstallMyAgent,Values=true" \
  --schedule-expression "rate(1 day)" \
  --parameters '{
    "action": ["Install"],
    "name": ["MyCompanyAgent"],
    "version": ["1.2.3"]
  }' \
  --association-name "EnsureMyCompanyAgent"
```

---

## AWS-Managed Packages

### Packages AWS Cung Cấp Sẵn

| Package | Mô Tả |
|---------|-------|
| `AmazonCloudWatchAgent` | CloudWatch Agent — thu thập metrics và logs |
| `AWSCodeDeployAgent` | CodeDeploy Agent — triển khai ứng dụng |
| `AWSSupport-EC2Rescue` | Công cụ chẩn đoán sự cố EC2 |
| `AmazonInspector2Agent` | Inspector Agent — quét vulnerability |
| `AWSPVDriver` | PV Driver cho Windows (AWS driver) |

```bash
# Cài CloudWatch Agent trên tất cả production instances
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{
    "action": ["Install"],
    "name": ["AmazonCloudWatchAgent"]
  }'
```

---

## OpsCenter — Trung Tâm Vận Hành

### OpsCenter Là Gì?

**OpsCenter** (Trung Tâm Vận Hành) là nơi tập trung để kỹ sư vận hành (DevOps/SRE) xem, điều tra và giải quyết các vấn đề vận hành (**OpsItems**) từ nhiều nguồn AWS khác nhau — CloudWatch, Config, Security Hub, EventBridge — trong một giao diện duy nhất.

### Kiến Trúc OpsCenter

```
Nhiều nguồn tạo OpsItems:
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  CloudWatch Alarm ─────┐                                     │
│  AWS Config Rule ──────┤                                     │
│  Security Hub ─────────┼──▶ EventBridge ──▶ OpsCenter       │
│  GuardDuty ────────────┤         │            │              │
│  Personal Health ──────┘         │            ▼             │
│  Lambda (custom) ─────────────────▶     OpsItems            │
│                                         │                   │
│                                         ▼                   │
│                               Ops Engineer                   │
│                               Xem, phân tích, xử lý         │
│                                         │                   │
│                                         ▼                   │
│                               Chạy SSM Automation           │
│                               (Remediation Runbook)          │
└──────────────────────────────────────────────────────────────┘
```

---

## OpsItems — Mục Vận Hành

### OpsItem Là Gì?

**OpsItem** (Mục Vận Hành) là một bản ghi mô tả vấn đề vận hành cần điều tra hoặc giải quyết. Tương tự như "ticket" trong ITSM (IT Service Management — Quản Lý Dịch Vụ IT) nhưng tích hợp sâu với AWS.

### Cấu Trúc OpsItem

```json
{
  "Title": "EC2 Instance CPU Utilization High",
  "Description": "Instance i-1234567890abcdef0 has CPU > 90% for 15 minutes",
  "Category": "Performance",
  "Severity": "2",
  "Status": "Open",
  "Source": "aws:cloudwatch",
  "OperationalData": {
    "/aws/resources": {
      "Value": "[{\"arn\": \"arn:aws:ec2:ap-southeast-1:123456789:instance/i-1234567890abcdef0\"}]",
      "Type": "SearchableString"
    },
    "/aws/automations": {
      "Value": "[{\"automationId\": \"AWS-RestartEC2Instance\"}]",
      "Type": "SearchableString"
    }
  },
  "Notifications": [
    {"Arn": "arn:aws:sns:ap-southeast-1:123456789:ops-team"}
  ]
}
```

### Severity Levels

| Severity | Ý Nghĩa | SLA Response |
|----------|---------|-------------|
| 1 — Critical | Ảnh hưởng production, mất doanh thu | < 15 phút |
| 2 — High | Ảnh hưởng chức năng quan trọng | < 1 giờ |
| 3 — Medium | Ảnh hưởng một phần | < 4 giờ |
| 4 — Low | Vấn đề nhỏ, không cấp bách | < 24 giờ |

### Tạo OpsItem Thủ Công / Lập Trình

```bash
# Tạo OpsItem từ CLI
aws ssm create-ops-item \
  --title "Disk usage > 90% on prod-db-01" \
  --description "Server prod-db-01 disk usage reached 92%. Immediate cleanup required." \
  --category "Availability" \
  --severity "2" \
  --source "custom-monitoring" \
  --notifications '[{"Arn":"arn:aws:sns:ap-southeast-1:123456789:ops-alerts"}]' \
  --operational-data '{
    "/aws/resources": {
      "Value": "[{\"arn\":\"arn:aws:ec2:ap-southeast-1:123456789:instance/i-1234567890abcdef0\"}]",
      "Type": "SearchableString"
    }
  }'
```

```python
# Tạo OpsItem từ Lambda (tích hợp EventBridge)
import boto3

def create_ops_item_from_alarm(event):
    ssm = boto3.client('ssm')
    
    alarm_name = event['detail']['alarmName']
    instance_id = event['detail']['configuration']['metrics'][0]['metricStat']['metric']['dimensions']['InstanceId']
    
    response = ssm.create_ops_item(
        Title=f'CloudWatch Alarm: {alarm_name}',
        Description=f'Alarm triggered for instance {instance_id}. Check CloudWatch for details.',
        Category='Performance',
        Severity='2',
        Source='aws:cloudwatch',
        OperationalData={
            '/aws/resources': {
                'Value': f'[{{"arn":"arn:aws:ec2:ap-southeast-1:123456789:instance/{instance_id}"}}]',
                'Type': 'SearchableString'
            },
            '/aws/automations': {
                'Value': '[{"automationId": "AWS-GatherDiagnosticsFromEC2"}]',
                'Type': 'SearchableString'
            }
        },
        Notifications=[
            {'Arn': 'arn:aws:sns:ap-southeast-1:123456789:ops-team'}
        ]
    )
    return response['OpsItemId']
```

### Quản Lý OpsItems

```bash
# Xem tất cả OpsItems đang mở
aws ssm describe-ops-items \
  --ops-item-filters "Key=Status,Values=Open,Operator=Equal" \
  --query 'OpsItemSummaries[*].[OpsItemId,Title,Severity,Status,CreatedTime]' \
  --output table

# Cập nhật trạng thái OpsItem
aws ssm update-ops-item \
  --ops-item-id "oi-abc123def456" \
  --status InProgress \
  --description "Investigating high CPU on prod instances. Collecting diagnostics."

# Đóng OpsItem sau khi giải quyết
aws ssm update-ops-item \
  --ops-item-id "oi-abc123def456" \
  --status Resolved \
  --description "Resolved: Memory leak in OrderService v2.3.1. Restarted service and deployed fix."
```

---

## Tích Hợp OpsCenter Với AWS Services

### EventBridge Rules Tự Động Tạo OpsItems

```json
// EventBridge Rule: CloudWatch Alarm → OpsCenter
{
  "source": ["aws.cloudwatch"],
  "detail-type": ["CloudWatch Alarm State Change"],
  "detail": {
    "state": {
      "value": ["ALARM"]
    }
  }
}

// Target: SSM CreateOpsItem
{
  "TargetArn": "arn:aws:ssm:ap-southeast-1::document/AWS-CreateOpsItem",
  "RoleArn": "arn:aws:iam::123456789:role/EventBridgeSSMRole",
  "Input": {
    "Title": "CloudWatch Alarm: <detail.alarmName>",
    "Category": "Performance",
    "Severity": "2",
    "Source": "aws:cloudwatch"
  }
}
```

### OpsCenter Insight Sources

| Nguồn | Loại OpsItems Tự Tạo |
|-------|---------------------|
| CloudWatch Alarms | CPU/Memory/Disk alerts |
| AWS Config | Compliance violations |
| Security Hub | Security findings |
| AWS Health | Service health events |
| Systems Manager (Patch) | Patch compliance failures |
| GuardDuty | Threat detections |
| Trusted Advisor | Best practice violations |

### Bật Tự Động Tạo OpsItems Từ CloudWatch

```bash
# Cấu hình CloudWatch Alarm để tự tạo OpsItem
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-ProdFleet" \
  --alarm-description "CPU > 90% on production instances" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions \
    "arn:aws:ssm:ap-southeast-1:123456789:opsitem:3" \
    "arn:aws:sns:ap-southeast-1:123456789:ops-team"
```

---

## Runbook Tự Động Từ OpsCenter

### Chạy Automation Từ OpsItem

Khi xem OpsItem trong console, có thể click nút "Run automation" để trigger SSM Automation document liên quan — không cần mở terminal.

```bash
# Liên kết Automation document với OpsItem
aws ssm update-ops-item \
  --ops-item-id "oi-abc123def456" \
  --operational-data '{
    "/aws/automations": {
      "Value": "[{\"automationId\": \"AWS-GatherDiagnosticsFromEC2\", \"automationVersion\": \"1\"}]",
      "Type": "SearchableString"
    }
  }'
```

### Workflow Điển Hình Với OpsCenter

```
1. CloudWatch Alarm: CPU > 90% trên i-1234567890abcdef0
         │
         ▼
2. EventBridge → Tự động tạo OpsItem (Severity 2)
         │
         ▼
3. SNS notification → Slack/Email cho on-call engineer
         │
         ▼
4. Engineer mở OpsItem trong OpsCenter
   → Xem resource details, alarm history
   → Click "Run automation": AWS-GatherDiagnosticsFromEC2
         │
         ▼
5. Automation thu thập: ps aux, memory, network connections
   → Kết quả hiển thị ngay trong OpsItem
         │
         ▼
6. Engineer phân tích → Phát hiện memory leak trong process "orderservice"
         │
         ▼
7. Run automation: AWS-RunShellScript → Restart orderservice
         │
         ▼
8. Verify: CloudWatch metrics trở lại bình thường
         │
         ▼
9. Update OpsItem status → Resolved
   → Ghi root cause và solution vào Description
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: SSM Distributor khác S3 + script thông thường như thế nào?**

> Distributor thêm: (1) Quản lý phiên bản (versioning) — dễ dàng rollback về version cũ; (2) Multi-OS support tự động — một package xử lý Amazon Linux, Ubuntu, Windows với install script phù hợp; (3) Tích hợp với State Manager — đảm bảo package luôn được cài trên toàn fleet; (4) Kiểm tra checksum để xác thực package tính toàn vẹn; (5) Audit trail — biết ai cài gì, khi nào.

**Q: OpsCenter giải quyết vấn đề gì? Tại sao không dùng PagerDuty/Jira?**

> OpsCenter giải quyết vấn đề **context switching** — kỹ sư vận hành phải mở nhiều tab (CloudWatch, Config, EC2 console, incident ticket) mới có đủ thông tin. OpsCenter tập hợp tất cả vào một nơi: resource details, alarm history, related OpsItems, và Automation runbooks. Tích hợp sâu với SSM cho phép chạy remediation trực tiếp từ OpsItem mà không cần copy-paste instance IDs. PagerDuty/Jira vẫn dùng cho alerting và project tracking, OpsCenter bổ sung context AWS-specific.

**Q: OpsItem khác CloudWatch Alarm như thế nào?**

> CloudWatch Alarm là ngưỡng cảnh báo — khi vượt ngưỡng, gửi notification và trigger action. OpsItem là bản ghi vấn đề cần điều tra — có trạng thái (Open/InProgress/Resolved), lịch sử thay đổi, người phụ trách, kết quả chạy automation. CloudWatch Alarm thường tạo OpsItem tự động — Alarm là trigger, OpsItem là nơi quản lý quá trình giải quyết sự cố.

### Nâng Cao

**Q: Thiết kế incident management platform sử dụng SSM OpsCenter và các AWS services khác?**

> **(1) Detection (Phát Hiện):** CloudWatch Alarms + Config Rules + GuardDuty → EventBridge → Tự động tạo OpsItems với severity, context, và related resources.
>
> **(2) Notification:** OpsItem Notifications → SNS → Lambda → Slack/PagerDuty. Severity 1 → PagerDuty page. Severity 2-3 → Slack channel. Severity 4 → Email digest.
>
> **(3) Triage (Phân Loại):** OpsCenter dashboard hiển thị tất cả Open OpsItems. Link đến CloudWatch dashboard, CloudTrail events, Config timeline của resource liên quan. Automation suggestions dựa trên OpsItem category.
>
> **(4) Investigation (Điều Tra):** Pre-built Automation runbooks cho common scenarios: "HighCPU-Diagnostics", "DiskFull-Cleanup", "OutOfMemory-Analysis". Kỹ sư click Run → Kết quả ghi vào OpsItem.
>
> **(5) Remediation (Khắc Phục):** Automation runbooks có approval gate (aws:approve) cho tác vụ nguy hiểm. Kết quả auto-remediation ghi vào OpsItem.
>
> **(6) Post-mortem (Tổng Kết):** OpsItem history lưu toàn bộ timeline, actions taken, kết quả. Export sang S3 → Athena → QuickSight dashboard cho trend analysis.

---

**Liên Quan:** [Session Manager](./1-session-manager.md) | [Run Command & Automation](./4-run-command-automation.md) | [Inventory & State Manager](./5-inventory-compliance.md)

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Module:** 04-systems-manager (6/6 files) — Hoàn Thành
