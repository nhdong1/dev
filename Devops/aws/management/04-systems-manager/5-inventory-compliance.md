# SSM Inventory & State Manager — Kiểm Kê Phần Mềm & Quản Lý Trạng Thái

> **SSM Inventory** (Kiểm Kê SSM) thu thập thông tin về phần mềm, cấu hình, và trạng thái của managed instances. **State Manager** (Trình Quản Lý Trạng Thái) đảm bảo instances luôn ở trạng thái mong muốn bằng cách liên tục áp dụng cấu hình và phát hiện drift (sai lệch so với cấu hình chuẩn).

---

## 📚 Mục Lục

1. [SSM Inventory — Kiểm Kê Hệ Thống](#ssm-inventory--kiểm-kê-hệ-thống)
2. [Loại Dữ Liệu Inventory](#loại-dữ-liệu-inventory)
3. [Custom Inventory](#custom-inventory)
4. [Truy Vấn & Phân Tích Inventory](#truy-vấn--phân-tích-inventory)
5. [SSM State Manager — Quản Lý Trạng Thái](#ssm-state-manager--quản-lý-trạng-thái)
6. [Associations — Liên Kết Cấu Hình](#associations--liên-kết-cấu-hình)
7. [Compliance Reporting](#compliance-reporting)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## SSM Inventory — Kiểm Kê Hệ Thống

### Inventory Là Gì?

**SSM Inventory** tự động thu thập dữ liệu từ managed instances theo lịch và lưu vào AWS Systems Manager Inventory data store (có thể sync sang S3 + Athena để query bằng SQL).

```
Luồng thu thập Inventory:
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Inventory Association (theo lịch)                         │
│         │                                                  │
│         ▼                                                  │
│  SSM Agent trên mỗi instance thu thập:                     │
│  • Application list (danh sách ứng dụng)                   │
│  • Network config (cấu hình mạng)                          │
│  • OS info (thông tin hệ điều hành)                        │
│  • Running services (dịch vụ đang chạy)                    │
│  • Custom inventory (dữ liệu tùy chỉnh)                    │
│         │                                                  │
│         ▼                                                  │
│  SSM Inventory Data Store                                  │
│         │                                                  │
│         ▼                                S3 Bucket         │
│  Resource Data Sync ──────────────────▶ Athena Query       │
│                                         QuickSight         │
└────────────────────────────────────────────────────────────┘
```

### Bật Inventory Cho Fleet

```bash
# Tạo Inventory Association cho tất cả managed instances
aws ssm create-association \
  --name "AWS-GatherSoftwareInventory" \
  --targets "Key=InstanceIds,Values=*" \
  --schedule-expression "rate(1 day)" \
  --parameters '{
    "applications": ["Enabled"],
    "awsComponents": ["Enabled"],
    "networkConfig": ["Enabled"],
    "windowsUpdates": ["Disabled"],
    "instanceDetailedInformation": ["Enabled"],
    "services": ["Enabled"],
    "windowsRegistry": ["Disabled"],
    "files": ["Enabled"],
    "customInventory": ["Enabled"]
  }' \
  --association-name "DailyInventoryCollection"
```

---

## Loại Dữ Liệu Inventory

### Inventory Types Chuẩn

| Inventory Type | Dữ Liệu Thu Thập | OS |
|---------------|-----------------|-----|
| `AWS:Application` | Tên, phiên bản, publisher của ứng dụng đã cài | Linux + Windows |
| `AWS:AWSComponent` | AWS agents và tools (CloudWatch Agent, CodeDeploy...) | Linux + Windows |
| `AWS:Network` | Địa chỉ IP, MAC address, DNS servers | Linux + Windows |
| `AWS:InstanceDetailedInformation` | CPU, RAM, OS version, kernel | Linux + Windows |
| `AWS:Service` | Danh sách services đang chạy | Linux + Windows |
| `AWS:WindowsUpdate` | Windows Update status và patches | Windows only |
| `AWS:WindowsRegistry` | Registry keys | Windows only |
| `AWS:File` | Metadata của files theo path pattern | Linux + Windows |
| `AWS:PatchSummary` | Kết quả patch compliance | Linux + Windows |

### Ví Dụ Dữ Liệu Inventory Thu Thập Được

```json
// AWS:Application — Ứng dụng cài trên instance
{
  "TypeName": "AWS:Application",
  "InstanceId": "i-1234567890abcdef0",
  "CaptureTime": "2026-05-17T10:00:00Z",
  "Content": [
    {
      "Name": "nginx",
      "Version": "1.24.0",
      "Publisher": "nginx.org",
      "InstalledTime": "2026-01-15T08:00:00Z"
    },
    {
      "Name": "java-17-amazon-corretto",
      "Version": "17.0.10",
      "Publisher": "Amazon.com Inc.",
      "InstalledTime": "2026-01-15T08:05:00Z"
    }
  ]
}

// AWS:InstanceDetailedInformation — Thông tin phần cứng
{
  "TypeName": "AWS:InstanceDetailedInformation",
  "Content": [
    {
      "CPUModel": "Intel(R) Xeon(R) Platinum 8259CL CPU @ 2.50GHz",
      "CPUCores": "2",
      "CPUSpeedMHz": "2499",
      "OSName": "Amazon Linux",
      "OSVersion": "2023",
      "KernelVersion": "6.1.82-99.169.amzn2023.x86_64",
      "MemorySizeMB": "3942"
    }
  ]
}
```

---

## Custom Inventory

### Custom Inventory Là Gì?

**Custom Inventory** cho phép bạn đưa bất kỳ dữ liệu nào vào SSM Inventory — ví dụ: thông tin deployment, license key, business metadata.

### Tạo Custom Inventory

```bash
# Bước 1: Tạo file inventory metadata trên instance
# File đặt tại: /var/lib/amazon/ssm/inventory/custom/
sudo mkdir -p /var/lib/amazon/ssm/inventory/custom

# Tạo file JSON theo định dạng SSM
sudo cat > /var/lib/amazon/ssm/inventory/custom/MyAppInfo.json << 'EOF'
{
  "SchemaVersion": "1.0",
  "TypeName": "Custom:MyAppDeployment",
  "Content": [
    {
      "AppName": "OrderService",
      "Version": "2.5.3",
      "DeployedBy": "ci-cd-pipeline",
      "DeployedAt": "2026-05-17T09:30:00Z",
      "GitCommit": "abc123def456",
      "Environment": "production",
      "Port": "8080"
    }
  ]
}
EOF
```

```bash
# Bước 2: Bật custom inventory trong Association
aws ssm put-inventory \
  --instance-id i-1234567890abcdef0 \
  --items '[{
    "TypeName": "Custom:MyAppDeployment",
    "SchemaVersion": "1.0",
    "CaptureTime": "2026-05-17T09:30:00Z",
    "Content": [{
      "AppName": "OrderService",
      "Version": "2.5.3",
      "DeployedBy": "ci-cd-pipeline",
      "GitCommit": "abc123def456"
    }]
  }]'
```

---

## Truy Vấn & Phân Tích Inventory

### Truy Vấn Từ CLI

```bash
# Tìm tất cả instances cài nginx
aws ssm get-inventory \
  --filters '[{
    "Key": "AWS:Application.Name",
    "Values": ["nginx"],
    "Type": "Equal"
  }]' \
  --result-attributes '[{"TypeName": "AWS:Application"}]' \
  --query 'Entities[*].[Id, Data."AWS:Application".Content[?Name==`nginx`].Version | [0][0]]' \
  --output table

# Tìm instances có Java version cũ (< 11)
aws ssm get-inventory \
  --filters '[{
    "Key": "AWS:Application.Name",
    "Values": ["java"],
    "Type": "Contains"
  }]'

# Xem inventory của một instance cụ thể
aws ssm list-inventory-entries \
  --instance-id i-1234567890abcdef0 \
  --type-name "AWS:Application" \
  --query 'Entries[*].[Name,Version]' \
  --output table
```

### Resource Data Sync — Đẩy Sang S3 Để Query Bằng Athena

```bash
# Tạo Resource Data Sync: đẩy inventory vào S3
aws ssm create-resource-data-sync \
  --sync-name "InventoryToS3" \
  --s3-destination '{
    "BucketName": "my-ssm-inventory-bucket",
    "Prefix": "inventory-data/",
    "SyncFormat": "JsonSerDe",
    "Region": "ap-southeast-1",
    "AWSKMSKeyARN": "arn:aws:kms:ap-southeast-1:123456789:key/abcd-1234"
  }'
```

```sql
-- Athena query: Tìm instances có phiên bản log4j lỗi
SELECT
  resourceid,
  json_extract_scalar(content, '$.Name') AS package_name,
  json_extract_scalar(content, '$.Version') AS version
FROM "ssm_inventory"."aws_application"
WHERE json_extract_scalar(content, '$.Name') LIKE '%log4j%'
  AND json_extract_scalar(content, '$.Version') < '2.17.0'
ORDER BY resourceid;

-- Tổng hợp: Bao nhiêu instances dùng mỗi phiên bản nginx
SELECT
  json_extract_scalar(content, '$.Version') AS nginx_version,
  COUNT(DISTINCT resourceid) AS instance_count
FROM "ssm_inventory"."aws_application"
WHERE json_extract_scalar(content, '$.Name') = 'nginx'
GROUP BY 1
ORDER BY instance_count DESC;
```

---

## SSM State Manager — Quản Lý Trạng Thái

### State Manager Là Gì?

**State Manager** (Trình Quản Lý Trạng Thái) đảm bảo managed instances **luôn ở trạng thái mong muốn** bằng cách tự động áp dụng cấu hình theo lịch và phát hiện/sửa **configuration drift** (sai lệch cấu hình).

### Vấn Đề Configuration Drift

```
Configuration Drift (Sai Lệch Cấu Hình):
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Trạng thái mong muốn (Desired State):                       │
│  ✅ CloudWatch Agent: Installed & Running                    │
│  ✅ Security patches: Up to date                             │
│  ✅ /etc/ssh/sshd_config: PermitRootLogin=no                 │
│  ✅ Nginx: version 1.24.x                                    │
│                                                              │
│  Sau 3 tháng, một instance bị drift:                         │
│  ❌ CloudWatch Agent: Removed (ai đó chạy yum remove)        │
│  ❌ Security patches: Missing 5 patches                      │
│  ❓ sshd_config: Ai đó đổi thành PermitRootLogin=yes        │
│  ❌ Nginx: Downgraded to 1.22.x                              │
│                                                              │
│  State Manager tự động phát hiện và sửa!                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Associations — Liên Kết Cấu Hình

### Association Là Gì?

**Association** (Liên Kết) là cấu hình nối SSM Document với targets (instances) và lịch chạy. State Manager dùng Associations để liên tục đảm bảo trạng thái mong muốn.

### Tạo Association

```bash
# Đảm bảo CloudWatch Agent luôn được cài và chạy
aws ssm create-association \
  --name "AWS-ConfigureAWSPackage" \
  --targets "Key=tag:Environment,Values=Production" \
  --schedule-expression "rate(1 day)" \
  --parameters '{
    "action": ["Install"],
    "name": ["AmazonCloudWatchAgent"],
    "version": ["latest"]
  }' \
  --association-name "EnsureCloudWatchAgent" \
  --compliance-severity CRITICAL \
  --apply-only-at-cron-interval false

# Association cấu hình SSH hardening
aws ssm create-association \
  --name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --schedule-expression "rate(12 hours)" \
  --parameters '{
    "commands": [
      "sed -i \"s/PermitRootLogin yes/PermitRootLogin no/g\" /etc/ssh/sshd_config",
      "sed -i \"s/PasswordAuthentication yes/PasswordAuthentication no/g\" /etc/ssh/sshd_config",
      "systemctl reload sshd",
      "echo SSH hardening applied: $(date)"
    ]
  }' \
  --association-name "SSHHardening"
```

### Association Compliance

```bash
# Xem compliance của tất cả associations
aws ssm list-associations \
  --association-filter-list "key=AssociationName,value=EnsureCloudWatchAgent"

# Xem instances NON_COMPLIANT với association cụ thể
aws ssm describe-association-executions \
  --association-id "12345678-1234-1234-1234-123456789012" \
  --filters "Key=Status,Value=Failed" \
  --query 'AssociationExecutions[*].[ExecutionId,Status,DetailedStatus,CreatedTime]'
```

### Ví Dụ Thực Tế: Đảm Bảo Tuân Thủ CIS Benchmark

```yaml
# Document để enforce CIS Level 1 settings
schemaVersion: "2.2"
description: "CIS Amazon Linux 2 Benchmark Level 1"
mainSteps:
  - action: aws:runShellScript
    name: disableUnusedFilesystems
    inputs:
      runCommand:
        - |
          # CIS 1.1.1 - Disable unused filesystems
          for fs in cramfs freevxfs jffs2 hfs hfsplus squashfs udf; do
            echo "install $fs /bin/true" >> /etc/modprobe.d/CIS.conf
            modprobe -r $fs 2>/dev/null || true
          done

  - action: aws:runShellScript
    name: configureSSH
    inputs:
      runCommand:
        - |
          # CIS 5.2 - SSH Server Configuration
          SSHD_CONFIG="/etc/ssh/sshd_config"
          
          grep -q "^Protocol" $SSHD_CONFIG || echo "Protocol 2" >> $SSHD_CONFIG
          sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' $SSHD_CONFIG
          sed -i 's/^#\?IgnoreRhosts.*/IgnoreRhosts yes/' $SSHD_CONFIG
          sed -i 's/^#\?HostbasedAuthentication.*/HostbasedAuthentication no/' $SSHD_CONFIG
          sed -i 's/^#\?PermitEmptyPasswords.*/PermitEmptyPasswords no/' $SSHD_CONFIG
          
          systemctl reload sshd
          echo "SSH hardening complete"

  - action: aws:runShellScript
    name: configureAuditd
    inputs:
      runCommand:
        - |
          # CIS 4.1 - Configure auditd
          systemctl enable auditd
          systemctl start auditd
          
          # Audit sudo usage
          echo "-a always,exit -F arch=b64 -S execve -C uid!=euid -F euid=0 -k sudo_log" \
            >> /etc/audit/rules.d/audit.rules
          
          augenrules --load
          echo "Audit daemon configured"
```

---

## Compliance Reporting

### SSM Compliance Dashboard

```
Systems Manager → Compliance
→ Xem tổng quan: Patch Compliance + Association Compliance
→ Drill down theo instance, patch group, association
```

### Xem Compliance Programmatically

```bash
# Compliance summary toàn fleet
aws ssm list-compliance-summaries \
  --query 'ComplianceSummaryItems[*].[ComplianceType,CompliantResourceCount.CriticalCount,NonCompliantResourceCount.CriticalCount]' \
  --output table

# Instances không compliant với patch (thiếu critical patches)
aws ssm list-resource-compliance-summaries \
  --filters "Key=ComplianceType,Values=Patch" "Key=OverallSeverity,Values=CRITICAL" \
  --query 'ResourceComplianceSummaryItems[?Status==`NON_COMPLIANT`].[ResourceId,OverallSeverity,Status]' \
  --output table

# Chi tiết từng patch bị thiếu trên một instance
aws ssm list-compliance-items \
  --resource-ids i-1234567890abcdef0 \
  --resource-types ManagedInstance \
  --filters "Key=ComplianceType,Values=Patch" "Key=Status,Values=NON_COMPLIANT" \
  --query 'ComplianceItems[*].[Id,Title,Severity,Status]' \
  --output table
```

### Tích Hợp Với AWS Config

```
SSM Patch Compliance → AWS Config
Config Rule: ec2-managedinstance-patch-compliance-status-check
→ Instances thiếu patches → NON_COMPLIANT trong Config
→ Config Aggregator tổng hợp compliance đa account
→ Security Hub nhận findings
```

### Tích Hợp Với Security Hub

```bash
# Tạo custom finding từ SSM Compliance vào Security Hub
import boto3

def report_noncompliant_to_security_hub(instance_id, patch_id, severity):
    securityhub = boto3.client('securityhub')
    
    securityhub.batch_import_findings(
        Findings=[{
            'SchemaVersion': '2018-10-08',
            'Id': f'{instance_id}/{patch_id}',
            'ProductArn': f'arn:aws:securityhub:{region}:{account}:product/{account}/default',
            'GeneratorId': 'SSM-Patch-Compliance',
            'AwsAccountId': account,
            'Types': ['Software and Configuration Checks/Patch Management'],
            'Severity': {'Label': severity},
            'Title': f'Missing security patch: {patch_id}',
            'Description': f'Instance {instance_id} is missing {severity} patch {patch_id}',
            'Resources': [{
                'Type': 'AwsEc2Instance',
                'Id': f'arn:aws:ec2:{region}:{account}:instance/{instance_id}'
            }]
        }]
    )
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: SSM Inventory thu thập những loại dữ liệu gì? Dùng để làm gì?**

> SSM Inventory thu thập: danh sách ứng dụng đã cài, network configuration, OS version, kernel version, services đang chạy, patch status. Dùng cho: (1) Audit phần mềm để tìm phiên bản lỗi (ví dụ Log4j cũ), (2) License compliance, (3) Phát hiện phần mềm không được phép, (4) Input cho Patch Manager biết instance cần patch gì.

**Q: State Manager khác Patch Manager như thế nào?**

> Patch Manager chuyên về vá lỗi OS và software packages. State Manager rộng hơn — đảm bảo bất kỳ cấu hình nào (SSH hardening, agent installation, firewall rules, file content) luôn ở trạng thái mong muốn. Patch Manager cũng dùng State Manager (Associations) để lên lịch patching. Tóm lại: Patch Manager là use case đặc biệt, State Manager là cơ chế chung.

**Q: Configuration drift là gì? Làm thế nào phát hiện và sửa?**

> Configuration drift xảy ra khi trạng thái thực tế của instance khác với trạng thái mong muốn — do thay đổi thủ công, lỗi script, hoặc incident response. State Manager phát hiện bằng cách chạy Association định kỳ và so sánh output. Nếu drift được phát hiện (ví dụ SSH config bị thay đổi), State Manager tự động chạy lại document để đưa về trạng thái đúng và đánh dấu NON_COMPLIANT cho audit.

### Nâng Cao

**Q: Thiết kế giải pháp phát hiện và sửa configuration drift cho 1000 EC2 production?**

> **(1) Baseline cấu hình:** Định nghĩa desired state qua SSM Documents — bao gồm: SSH hardening, CloudWatch Agent, security patches, audit daemon, kernel parameters.
>
> **(2) State Manager Associations:** Tạo Associations chạy mỗi 12-24 giờ trên toàn fleet. `apply-only-at-cron-interval=false` đảm bảo áp dụng ngay khi instance mới join fleet.
>
> **(3) Compliance reporting:** Cấu hình `ComplianceSeverity=CRITICAL` cho security-related associations. Kết nối với Config Rule và Security Hub để có dashboard tập trung.
>
> **(4) Alerting:** CloudWatch alarm khi `SSM:ComplianceItemsNonCompliant` > ngưỡng. SNS notification đến ops team.
>
> **(5) Inventory correlation:** Dùng Resource Data Sync đẩy inventory vào S3. Athena query hàng tuần tìm phần mềm lỗi thời (ví dụ: `SELECT * WHERE log4j version < 2.17.0`).
>
> **(6) Immutable infrastructure:** Kết hợp với Patch Manager để tạo AMI đã patch, dùng ASG Instance Refresh — thay vì patch in-place. Đây là cách tiếp cận modern nhất để tránh drift.

---

**Liên Quan:** [Patch Manager](./2-patch-manager.md) | [Run Command & Automation](./4-run-command-automation.md) | [Distributor & OpsCenter](./6-distributor-opscenter.md)
