# AWS Systems Manager — Quản Lý Hệ Thống Không Cần SSH

> AWS Systems Manager (SSM — Trình Quản Lý Hệ Thống AWS) là dịch vụ quản lý tập trung cho EC2 instances và on-premises servers. SSM cho phép quản trị server mà không cần mở port SSH (22) hoặc RDP (3389), tăng bảo mật đáng kể và đơn giản hóa vận hành.

---

## 📚 Mục Lục

1. [Tổng Quan SSM](#tổng-quan-ssm)
2. [SSM Session Manager](#ssm-session-manager)
3. [SSM Run Command](#ssm-run-command)
4. [SSM Patch Manager](#ssm-patch-manager)
5. [SSM Parameter Store](#ssm-parameter-store)
6. [SSM Inventory & Compliance](#ssm-inventory--compliance)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan SSM

### Kiến Trúc SSM

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Systems Manager                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Session   │  │     Run     │  │     Patch Manager        │ │
│  │   Manager   │  │   Command   │  │  (Quản Lý Bản Vá)       │ │
│  │  (Phiên Làm │  │  (Lệnh Từ  │  │                          │ │
│  │   Việc Từ  │  │   Xa)       │  │                          │ │
│  │   Xa)       │  │             │  │                          │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────────┘ │
│         │                │                     │                 │
│         └────────────────┼─────────────────────┘                │
│                          │                                       │
│              ┌───────────▼───────────┐                          │
│              │      SSM Agent         │                          │
│              │  (cài trên EC2/server) │                          │
│              └───────────────────────┘                          │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │  Parameter  │  │  Inventory   │  │   Automation            │ │
│  │   Store     │  │  (Kiểm Kê)   │  │   (Tự Động Hóa)        │ │
│  │  (Kho Tham  │  │             │  │                          │ │
│  │   Số)       │  │             │  │                          │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Cách SSM Agent Hoạt Động

```
EC2 Instance                    AWS SSM Service
─────────────                   ───────────────
SSM Agent               HTTPS   SSM Endpoints
(chạy dưới         ─────────►  (ssm.region.amazonaws.com)
 background)        polling

→ Agent KHÔNG cần inbound port mở
→ Agent tự kết nối ra ngoài (outbound HTTPS 443)
→ Dùng IAM Role của Instance Profile để xác thực
```

### Yêu Cầu Để Dùng SSM

```
1. SSM Agent được cài đặt (mặc định trên Amazon Linux 2, Windows Server 2019+)
2. EC2 Instance có IAM Role với SSM permissions:
   - AmazonSSMManagedInstanceCore (AWS Managed Policy)
3. Kết nối internet hoặc VPC Endpoint đến SSM service
4. (Optional) SSM VPC Endpoints để traffic không ra ngoài internet
```

```bash
# Kiểm tra SSM Agent status
sudo systemctl status amazon-ssm-agent

# Khởi động lại SSM Agent
sudo systemctl restart amazon-ssm-agent

# Xem version
amazon-ssm-agent --version

# Cài đặt thủ công trên Amazon Linux
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
```

### IAM Role Tối Thiểu Cho SSM

```json
// Gắn AWS Managed Policy này vào EC2 Instance Profile
{
  "PolicyArn": "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

// Bao gồm permissions:
// - ssm:DescribeAssociation
// - ssm:GetDocument
// - ssm:DescribeDocument
// - ssm:GetManifest
// - ssm:ListAssociations
// - ssm:UpdateAssociationStatus
// - ssm:UpdateInstanceAssociationStatus
// - ssm:UpdateInstanceInformation
// - ssmmessages:* (cho Session Manager)
// - ec2messages:* (cho Run Command)
```

---

## SSM Session Manager

### Tại Sao Dùng Session Manager Thay SSH?

```
Vấn Đề Với SSH (Secure Shell — Giao Thức Kết Nối Bảo Mật):
├── Phải mở port 22 → tấn công brute force, scanning
├── Cần quản lý SSH keys cho từng user
├── Khó audit: ai đã làm gì trong session?
├── Không hoạt động khi EC2 trong private subnet
└── Khi key bị mất → phải stop/start instance để reset

Lợi Ích Session Manager:
├── Không cần mở port — kết nối qua HTTPS (443)
├── Không cần SSH keys — dùng IAM để authenticate
├── Mọi session đều được log vào CloudWatch/S3
├── Hoạt động với instance trong private subnet
└── Có thể restrict dùng IAM policies chi tiết
```

### Thiết Lập Session Manager

```bash
# 1. Cài AWS CLI Plugin cho Session Manager
# (để dùng aws ssm start-session từ local terminal)
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm

# 2. Bắt đầu session từ local terminal
aws ssm start-session \
  --target i-1234567890abcdef0

# 3. Port forwarding qua Session Manager (không cần SSH tunnel)
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3306"],"localPortNumber":["13306"]}'
# → MySQL trên EC2 có thể truy cập qua localhost:13306

# 4. Kết nối SSH qua Session Manager tunnel (nếu vẫn cần SSH)
# ~/.ssh/config
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

### Cấu Hình Session Logging

```bash
# Cấu hình SSM Session Manager để log vào S3 và CloudWatch
aws ssm update-document \
  --name "SSM-SessionManagerRunShell" \
  --document-type "Session" \
  --content '{
    "schemaVersion": "1.0",
    "description": "Session Manager configuration",
    "sessionType": "Standard_Stream",
    "inputs": {
      "s3BucketName": "my-session-logs-bucket",
      "s3KeyPrefix": "ssm-sessions/",
      "s3EncryptionEnabled": true,
      "cloudWatchLogGroupName": "/aws/ssm/sessions",
      "cloudWatchEncryptionEnabled": true,
      "idleSessionTimeout": "30",
      "kmsKeyId": "arn:aws:kms:ap-southeast-1:123456789012:key/..."
    }
  }'
```

### IAM Policy Để Restrict Session Manager

```json
// Cho phép chỉ kết nối vào instances có tag Environment=Production
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ssm:resourceTag/Environment": "Production"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": "arn:aws:ssm:*:*:document/AWS-StartPortForwardingSession"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
    }
  ]
}
```

### Xem Session History

```bash
# Xem danh sách sessions
aws ssm describe-sessions \
  --state Active

aws ssm describe-sessions \
  --state History

# Terminate session đang chạy
aws ssm terminate-session \
  --session-id session-id-here

# Query CloudWatch Logs cho session logs
aws logs filter-log-events \
  --log-group-name /aws/ssm/sessions \
  --start-time $(date -d '-1 day' +%s000)
```

---

## SSM Run Command

### Khái Niệm Cơ Bản

```
Run Command (Lệnh Từ Xa):
└── Chạy scripts hoặc commands trên một hoặc nhiều EC2 instances
    mà không cần kết nối SSH

SSM Documents (Tài Liệu SSM):
└── Định nghĩa actions để thực hiện
    ├── AWS-RunShellScript — chạy shell script trên Linux
    ├── AWS-RunPowerShellScript — chạy PowerShell trên Windows
    ├── AWS-UpdateSSMAgent — cập nhật SSM Agent
    ├── AWS-RunAnsiblePlaybook — chạy Ansible playbook
    └── Custom Documents — tự tạo cho use case đặc biệt
```

### Ví Dụ Thực Tế

```bash
# Chạy command trên single instance
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=instanceIds,Values=i-1234567890abcdef0" \
  --parameters '{"commands":["df -h","free -m","ps aux | grep nginx"]}' \
  --output text

# Chạy command trên nhiều instances cùng tag
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters '{"commands":["sudo systemctl status nginx"]}' \
  --max-concurrency "50%" \  # Chạy tối đa 50% instances cùng lúc
  --max-errors "10%"         # Dừng nếu 10% fail

# Xem kết quả
COMMAND_ID=$(aws ssm send-command ... --query 'Command.CommandId' --output text)

aws ssm list-command-invocations \
  --command-id $COMMAND_ID \
  --details

aws ssm get-command-invocation \
  --command-id $COMMAND_ID \
  --instance-id i-1234567890abcdef0
```

### Custom SSM Document

```json
{
  "schemaVersion": "2.2",
  "description": "Deploy application update",
  "parameters": {
    "AppVersion": {
      "type": "String",
      "description": "Version to deploy"
    }
  },
  "mainSteps": [
    {
      "action": "aws:runShellScript",
      "name": "StopApp",
      "inputs": {
        "runCommand": [
          "sudo systemctl stop myapp"
        ]
      }
    },
    {
      "action": "aws:runShellScript",
      "name": "DeployApp",
      "inputs": {
        "runCommand": [
          "aws s3 cp s3://my-app-bucket/releases/{{AppVersion}}/app.tar.gz /tmp/",
          "cd /opt/myapp && sudo tar -xzf /tmp/app.tar.gz",
          "sudo systemctl start myapp",
          "sudo systemctl status myapp"
        ]
      }
    }
  ]
}
```

### State Manager — Quản Lý Trạng Thái

```bash
# State Manager giữ cho EC2 instances luôn ở trạng thái mong muốn
# Ví dụ: Luôn đảm bảo CloudWatch agent đang chạy

aws ssm create-association \
  --name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Role,Values=WebServer" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}' \
  --schedule-expression "rate(7 days)" \  # Kiểm tra mỗi 7 ngày
  --compliance-severity "MEDIUM"
```

---

## SSM Patch Manager

### Tầm Quan Trọng Của Patch Management

```
Unpatched Vulnerabilities (Lỗ Hổng Chưa Vá):
└── Nguyên nhân hàng đầu của security breaches

Thống kê:
• 60% breaches khai thác lỗ hổng đã có patch sẵn
• Trung bình 15 ngày từ khi patch ra đến khi bị khai thác
• EC2 instances cần patch OS, applications, và dependencies
```

### Khái Niệm Patch Manager

```
Patch Baseline (Đường Cơ Sở Vá Lỗi):
└── Định nghĩa LOẠI patches nào được phép install
    ├── Severity: Critical, Important, Medium, Low
    ├── Classification: Security, Bug Fix, Enhancement...
    └── Auto-approval delay: 7 ngày sau khi patch released

Patch Group (Nhóm Vá):
└── Gom nhóm instances cần cùng một Patch Baseline
    → Dùng tag "Patch Group" trên EC2 instances

Maintenance Window (Cửa Sổ Bảo Trì):
└── Khung thời gian cho phép patch được áp dụng
    → Tránh patch trong giờ cao điểm
```

### Thiết Lập Patch Manager

```bash
# Bước 1: Tạo Patch Baseline (Đường Cơ Sở Vá Lỗi)
aws ssm create-patch-baseline \
  --name "MyLinuxPatchBaseline" \
  --operating-system "AMAZON_LINUX_2" \
  --approval-rules '{
    "PatchRules": [
      {
        "PatchFilterGroup": {
          "PatchFilters": [
            {
              "Key": "SEVERITY",
              "Values": ["Critical", "Important"]
            },
            {
              "Key": "CLASSIFICATION",
              "Values": ["Security"]
            }
          ]
        },
        "ApproveAfterDays": 7,    // Auto-approve sau 7 ngày
        "ComplianceLevel": "HIGH"
      }
    ]
  }' \
  --description "Baseline for production Linux servers"

# Bước 2: Tạo Maintenance Window (Cửa Sổ Bảo Trì)
aws ssm create-maintenance-window \
  --name "WeeklyPatchWindow" \
  --schedule "cron(0 2 ? * SUN *)" \  # 2:00 AM mỗi Chủ Nhật
  --duration 4 \                       # Kéo dài tối đa 4 giờ
  --cutoff 1 \                          # Dừng task mới 1 giờ trước khi hết
  --allow-unassociated-targets false

# Bước 3: Thêm targets vào Maintenance Window
aws ssm register-target-with-maintenance-window \
  --window-id "mw-0a1b2c3d4e5f6g7h8" \
  --targets "Key=tag:PatchGroup,Values=production-linux" \
  --resource-type "INSTANCE"

# Bước 4: Đăng ký Patch task
aws ssm register-task-with-maintenance-window \
  --window-id "mw-0a1b2c3d4e5f6g7h8" \
  --targets "Key=WindowTargetIds,Values=target-id" \
  --task-arn "arn:aws:ssm:::document/AWS-RunPatchBaseline" \
  --task-type "RUN_COMMAND" \
  --task-invocation-parameters '{
    "RunCommand": {
      "Parameters": {
        "Operation": ["Install"]
      },
      "TimeoutSeconds": 3600
    }
  }' \
  --max-concurrency "20%" \
  --max-errors "5%"
```

### Patch Compliance Report — Báo Cáo Tuân Thủ Vá Lỗi

```bash
# Xem patch compliance của tất cả instances
aws ssm list-compliance-summaries \
  --filters "Key=ComplianceType,Type=EQUAL,Values=Patch"

# Xem chi tiết cho một instance
aws ssm list-compliance-items \
  --resource-types "ManagedInstance" \
  --resource-id "i-1234567890abcdef0" \
  --filters "Key=ComplianceType,Type=EQUAL,Values=Patch"

# Kết quả mẫu:
# {
#   "Id": "CVE-2023-1234",
#   "Title": "Security patch for OpenSSL",
#   "Severity": "CRITICAL",
#   "State": "NON_COMPLIANT",  ← Chưa được patch!
#   "InstalledTime": null
# }
```

---

## SSM Parameter Store

> Lưu ý: Parameter Store được đề cập chi tiết trong [3-secrets-management.md](./3-secrets-management.md). Đây là phần tóm tắt trong context của SSM.

### Parameter Store Trong SSM Ecosystem

```
Parameter Store (Kho Tham Số):
├── String — lưu cấu hình thông thường (database URL, feature flags)
├── StringList — danh sách giá trị phân cách bằng dấu phẩy
└── SecureString — lưu secrets (mật khẩu, API keys) với KMS encryption

Phân Cấp (Hierarchy):
/production/myapp/database/host
/production/myapp/database/port
/staging/myapp/database/host
```

```bash
# Lưu tham số
aws ssm put-parameter \
  --name "/production/myapp/db-host" \
  --value "mydb.cluster.ap-southeast-1.rds.amazonaws.com" \
  --type "String" \
  --tier "Standard"

# Lấy tham số trong EC2 User Data hoặc application
DB_HOST=$(aws ssm get-parameter \
  --name "/production/myapp/db-host" \
  --query "Parameter.Value" \
  --output text)

# Lấy nhiều tham số theo path
aws ssm get-parameters-by-path \
  --path "/production/myapp/" \
  --recursive \
  --with-decryption
```

---

## SSM Inventory & Compliance

### SSM Inventory — Kiểm Kê Hệ Thống

```bash
# SSM Inventory thu thập metadata về instances
# Bao gồm: OS, software, network config, running services...

# Kích hoạt Inventory collection
aws ssm create-association \
  --name "AWS-GatherSoftwareInventory" \
  --targets "Key=InstanceIds,Values=*" \
  --schedule-expression "rate(1 day)"

# Query inventory qua AWS Console hoặc API
aws ssm list-inventory-entries \
  --instance-id i-1234567890abcdef0 \
  --type-name "AWS:Application"

# Aggregate inventory data với Resource Data Sync → S3 → Athena
aws ssm create-resource-data-sync \
  --sync-name "AllInstancesInventory" \
  --s3-destination '{
    "BucketName": "my-inventory-bucket",
    "SyncFormat": "JsonSerDe",
    "Region": "ap-southeast-1"
  }'
```

### Dùng Systems Manager Fleet Manager

```
Fleet Manager (Quản Lý Đội Máy):
└── Giao diện trực quan trong AWS Console để:
    ├── Xem trạng thái tất cả managed instances
    ├── Browse file system từ xa (không cần SSH)
    ├── Xem running processes và performance metrics
    ├── Quản lý Windows Registry từ xa
    └── Kết nối RDP đến Windows instances (qua browser!)
```

---

## SSM Automation — Tự Động Hóa

### Automation Document Ví Dụ

```yaml
# Tự động tạo AMI sau khi patch thành công
description: "Patch và tạo AMI"
schemaVersion: "0.3"
assumeRole: "{{ AutomationAssumeRole }}"
parameters:
  InstanceId:
    type: String
  AMIName:
    type: String
mainSteps:
  - name: PatchInstance
    action: "aws:runCommand"
    inputs:
      DocumentName: AWS-RunPatchBaseline
      InstanceIds: ["{{ InstanceId }}"]
      Parameters:
        Operation: Install
  - name: StopInstance
    action: "aws:changeInstanceState"
    inputs:
      InstanceIds: ["{{ InstanceId }}"]
      DesiredState: stopped
  - name: CreateAMI
    action: "aws:createImage"
    inputs:
      InstanceId: "{{ InstanceId }}"
      ImageName: "{{ AMIName }}-{{ global:DATE }}"
      NoReboot: false
  - name: StartInstance
    action: "aws:changeInstanceState"
    inputs:
      InstanceIds: ["{{ InstanceId }}"]
      DesiredState: running
```

---

## So Sánh: SSH vs Session Manager

| Tiêu Chí | SSH (Secure Shell) | SSM Session Manager |
|----------|-------------------|---------------------|
| **Port** | 22 phải mở | Không cần port mở |
| **Authentication** | SSH Key Pairs | IAM Policies |
| **Audit** | Khó trace | CloudTrail + CloudWatch Logs |
| **Private Subnet** | Cần Bastion Host | Hoạt động trực tiếp |
| **Key Management** | Phức tạp | Không cần |
| **Cost** | Miễn phí | Miễn phí (SSM basic) |
| **Compliance** | Khó enforce | Dễ enforce qua IAM |
| **Setup** | Đơn giản | Cần SSM Agent + IAM Role |

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao nên dùng SSM Session Manager thay vì SSH để quản lý EC2?

**Trả lời:** Session Manager loại bỏ hoàn toàn nhu cầu mở port 22 (SSH) — đây là một trong những attack surface (bề mặt tấn công) phổ biến nhất. Thay vì dùng SSH keys (thường bị mất hoặc bị chia sẻ không đúng cách), Session Manager dùng IAM để authenticate và authorize, cho phép kiểm soát ai được kết nối vào instance nào với điều kiện gì. Mọi session đều được log tự động vào CloudWatch Logs và S3, đáp ứng yêu cầu audit compliance. Ngoài ra, Session Manager hoạt động với instances trong private subnet mà không cần Bastion Host (máy chủ trung gian), đơn giản hóa kiến trúc mạng.

### Q2: Patch Manager hoạt động như thế nào và khi nào dùng?

**Trả lời:** Patch Manager tự động hóa quá trình vá lỗi cho EC2 instances. Bạn tạo Patch Baseline (Đường Cơ Sở Vá Lỗi) định nghĩa loại patches nào được phép — theo severity (Critical, Important) và classification (Security, Bug Fix). Định nghĩa Maintenance Window (Cửa Sổ Bảo Trì) để patch xảy ra vào thời gian ít ảnh hưởng nhất (ví dụ 2 AM Chủ Nhật). Dùng Patch Groups (tag "Patch Group" trên instances) để áp dụng baselines khác nhau cho dev, staging, production. Sau đó có thể xem Patch Compliance Reports để biết instance nào chưa được vá. Nên dùng cho mọi EC2 fleet vì un-patched instances là nguyên nhân hàng đầu của breaches.

### Q3: SSM Parameter Store khác Secrets Manager ở điểm nào?

**Trả lời:** Đây là câu hỏi về trade-offs (đánh đổi). Parameter Store miễn phí cho Standard tier (max 4KB per parameter) và phù hợp cho configuration data (database URLs, feature flags, API endpoints không nhạy cảm). Secrets Manager tính phí $0.40/secret/tháng nhưng hỗ trợ automatic rotation (xoay vòng tự động) — điều này rất quan trọng cho database passwords và API keys. Secrets Manager tích hợp tốt hơn với RDS, Redshift, DocumentDB cho rotation. Nếu chỉ cần lưu secrets đơn giản không cần rotation, Parameter Store SecureString là lựa chọn kinh tế. Nếu cần rotation tự động, chọn Secrets Manager.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← IAM & Instance Profiles | [1-iam-instance-profiles.md](./1-iam-instance-profiles.md) |
| → Secrets Management | [3-secrets-management.md](./3-secrets-management.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
