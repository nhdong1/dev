# Change Audit — Kiểm Tra Ai Thay Đổi Gì, Khi Nào

> Change Audit — Kiểm Tra Thay Đổi — là quá trình ghi lại và truy vết mọi thay đổi hạ tầng: ai apply, thay đổi gì, khi nào, từ đâu. Đây là nền tảng của incident response — xử lý sự cố — và compliance — tuân thủ quy định.

---

## 🎯 Tại Sao Cần Audit Trail?

### Tình Huống Không Có Audit Trail

```
Thứ Hai, 9:00 sáng: Hệ thống production bắt đầu chậm bất thường
  
  Câu hỏi cần trả lời:
  ❓ Hạ tầng có thay đổi gì trong 24h qua không?
  ❓ Ai là người cuối cùng apply Terraform?
  ❓ Tại sao security group lại thay đổi?
  ❓ Lúc nào thì database connection pool bị thay đổi?
  
  Không có audit trail → mất 4-8 giờ để điều tra
  → Kéo dài MTTR — Mean Time To Recovery — Thời Gian Phục Hồi Trung Bình
```

### Với Audit Trail Đầy Đủ

```
Thứ Hai, 9:05 sáng:
  
  Kiểm tra audit log:
  ✅ 2026-05-11 23:45 UTC | user: ci-bot | action: apply
     Changed: aws_db_instance.prod — max_connections: 100 → 50
  
  Root cause tìm thấy trong 5 phút
  Fix và recovery trong 15 phút
  
  MTTR: 20 phút thay vì 4-8 giờ
```

---

## 📋 Các Nguồn Audit Data

### 1. Terraform State History — Lịch Sử State

Mỗi lần `terraform apply` tạo ra một phiên bản mới của state file (nếu dùng remote backend có versioning).

**AWS S3 + Versioning:**
```bash
# Xem tất cả phiên bản của state file
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix env/prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Output:
# --------------------------------------------------------------------------------------
# |                              ListObjectVersions                                     |
# +----------------------------------------------+-------------------------------------+
# |  vr8Abc123XYZ                                | 2026-05-12T10:30:00.000Z            |
# |  vr7Def456ABC                                | 2026-05-11T23:45:00.000Z            |
# |  vr6Ghi789DEF                                | 2026-05-10T14:20:00.000Z            |
# +----------------------------------------------+-------------------------------------+

# So sánh hai phiên bản state để thấy thay đổi gì
aws s3api get-object \
  --bucket my-terraform-state \
  --key env/prod/terraform.tfstate \
  --version-id vr7Def456ABC \
  state-before.json

aws s3api get-object \
  --bucket my-terraform-state \
  --key env/prod/terraform.tfstate \
  --version-id vr8Abc123XYZ \
  state-after.json

# Diff để thấy thay đổi
diff \
  <(jq '.resources | sort_by(.name)' state-before.json) \
  <(jq '.resources | sort_by(.name)' state-after.json)
```

### 2. AWS CloudTrail — Nhật Ký Hành Động API

CloudTrail ghi lại mọi API call đến AWS, kể cả những gì Terraform thực hiện.

**Bật CloudTrail cho toàn bộ region:**

```hcl
# cloudtrail.tf
resource "aws_cloudtrail" "main" {
  name                          = "terraform-audit-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true  # Bắt tất cả region
  enable_log_file_validation    = true  # Verify log integrity
  
  event_selector {
    read_write_type           = "WriteOnly"  # Chỉ ghi lại write actions
    include_management_events = true
    
    # Track cả S3 data events (ai đọc/sửa state file)
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::my-terraform-state/*"]
    }
  }

  tags = {
    ManagedBy = "terraform"
    Purpose   = "audit"
  }
}
```

**Query CloudTrail để tìm ai apply Terraform:**

```bash
# Tìm tất cả Terraform applies trong 24h qua
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=terraform-ci-role \
  --start-time "$(date -d '24 hours ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
  --query 'Events[*].[EventTime,Username,EventName,Resources[0].ResourceName]' \
  --output table

# Tìm thay đổi Security Group cụ thể
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --start-time "2026-05-11T00:00:00Z" \
  --end-time "2026-05-12T00:00:00Z"
```

**Dùng CloudWatch Logs Insights để query phức tạp:**

```sql
-- Query CloudTrail logs trong CloudWatch Logs
fields @timestamp, userIdentity.arn, eventName, requestParameters
| filter eventSource = "ec2.amazonaws.com"
  AND eventName in ["CreateSecurityGroup", "AuthorizeSecurityGroupIngress",
                    "RevokeSecurityGroupIngress", "DeleteSecurityGroup"]
| sort @timestamp desc
| limit 100
```

### 3. Terraform Cloud Audit Logs

Nếu dùng Terraform Cloud (HCP Terraform), có audit logs tích hợp:

```bash
# Lấy audit logs từ Terraform Cloud API
curl -H "Authorization: Bearer $TFC_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-org/audit-trail?since=2026-05-11T00:00:00Z" \
  | jq '.data[] | {
      time: .attributes.created_at,
      action: .attributes.action,
      resource: .attributes.resource.type,
      user: .attributes.auth.accessor_id
    }'

# Output mẫu:
# {
#   "time": "2026-05-11T23:45:12Z",
#   "action": "applied",
#   "resource": "run",
#   "user": "ci-service-account"
# }
```

### 4. Git History — Lịch Sử Git

Git là audit trail tự nhiên cho Terraform code:

```bash
# Ai thay đổi file nào, khi nào
git log --all --follow --format="%h %ai %an %s" -- infrastructure/prod/main.tf

# Xem nội dung thay đổi của commit cụ thể
git show abc1234 -- infrastructure/prod/main.tf

# Tìm commit nào đổi instance_type
git log -S 'instance_type' --all -- infrastructure/

# Blame — ai viết dòng code này
git blame infrastructure/prod/main.tf | grep instance_type
```

### 5. CI/CD Pipeline Logs

```yaml
# Trong GitHub Actions, lưu metadata vào apply output
- name: Terraform Apply với Metadata
  run: |
    echo "=== Apply Metadata ===" 
    echo "Triggered by: ${{ github.actor }}"
    echo "Commit: ${{ github.sha }}"
    echo "Branch: ${{ github.ref_name }}"
    echo "Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    echo "==="
    
    terraform apply -auto-approve -input=false tfplan
  env:
    TF_LOG: INFO
```

---

## 🏗️ Xây Dựng Hệ Thống Audit Toàn Diện

### Architecture Đề Xuất

```
                    Terraform Apply
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
    Git Commit      CI/CD Log      CloudTrail
    (code diff)   (who, when)   (AWS API calls)
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                   SIEM / Log Aggregator
                   (Splunk / Datadog / OpenSearch)
                         │
                         ▼
                   Audit Dashboard
                   + Alerting Rules
```

### Terraform Tags Cho Audit Context

```hcl
# Thêm metadata vào mọi resource qua default_tags
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy         = "terraform"
      TerraformWorkspace = terraform.workspace
      GitRepo           = "github.com/company/infrastructure"
      # Sử dụng TF_VAR để truyền từ CI/CD
      LastAppliedBy     = var.applied_by      # "github-actions" hoặc username
      LastAppliedCommit = var.git_commit_sha
      Environment       = var.environment
    }
  }
}

variable "applied_by" {
  description = "Who or what triggered this apply"
  default     = "unknown"
}

variable "git_commit_sha" {
  description = "Git commit SHA of the Terraform code"
  default     = "unknown"
}
```

**Trong CI/CD, truyền giá trị này:**

```bash
# Trong GitHub Actions
terraform apply \
  -var="applied_by=github-actions/${{ github.actor }}" \
  -var="git_commit_sha=${{ github.sha }}"
```

Kết quả: Mọi AWS resource sẽ có tags như:
```
LastAppliedBy:     github-actions/john.doe
LastAppliedCommit: abc123def456
Environment:       production
ManagedBy:         terraform
```

### SIEM Integration — Tích Hợp Hệ Thống Quản Lý Sự Kiện Bảo Mật

**Datadog Integration:**

```hcl
# Gửi CloudTrail events đến Datadog
resource "aws_cloudwatch_log_subscription_filter" "datadog_terraform" {
  name            = "terraform-audit-to-datadog"
  log_group_name  = aws_cloudwatch_log_group.cloudtrail.name
  filter_pattern  = "" # Tất cả events
  destination_arn = aws_lambda_function.datadog_forwarder.arn
}
```

**Tạo Dashboard trong Datadog/Grafana:**

```json
// Datadog query để đếm Terraform applies theo ngày
{
  "query": "source:cloudtrail @evt.name:terraform AND @usr.role:*ci*",
  "groupBy": ["date"],
  "aggregate": "count"
}
```

---

## 📊 Change Audit Report — Báo Cáo Thay Đổi

### Script Tạo Báo Cáo Hàng Tuần

```python
#!/usr/bin/env python3
"""
Script tạo báo cáo thay đổi Terraform hàng tuần
"""
import boto3
import json
from datetime import datetime, timedelta

def get_terraform_changes(days=7):
    """Lấy tất cả thay đổi Terraform trong N ngày qua"""
    
    cloudtrail = boto3.client('cloudtrail', region_name='ap-southeast-1')
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=days)
    
    # Events liên quan đến Terraform
    terraform_events = []
    paginator = cloudtrail.get_paginator('lookup_events')
    
    for page in paginator.paginate(
        LookupAttributes=[
            {
                'AttributeKey': 'Username',
                'AttributeValue': 'terraform-ci-role'
            }
        ],
        StartTime=start_time,
        EndTime=end_time
    ):
        terraform_events.extend(page['Events'])
    
    return terraform_events

def generate_report(events):
    """Tạo báo cáo từ events"""
    
    report = {
        'period': f"{datetime.utcnow() - timedelta(days=7)} → {datetime.utcnow()}",
        'total_changes': len(events),
        'changes_by_day': {},
        'changes_by_resource_type': {},
        'significant_changes': []
    }
    
    for event in events:
        # Nhóm theo ngày
        day = event['EventTime'].strftime('%Y-%m-%d')
        report['changes_by_day'][day] = report['changes_by_day'].get(day, 0) + 1
        
        # Nhóm theo loại resource
        event_name = event.get('EventName', 'Unknown')
        report['changes_by_resource_type'][event_name] = \
            report['changes_by_resource_type'].get(event_name, 0) + 1
        
        # Đánh dấu thay đổi quan trọng (security-related)
        security_events = [
            'AuthorizeSecurityGroupIngress',
            'CreateRole', 
            'AttachRolePolicy',
            'ModifyDBInstance'
        ]
        if event_name in security_events:
            report['significant_changes'].append({
                'time': event['EventTime'].isoformat(),
                'action': event_name,
                'user': event.get('Username', 'unknown')
            })
    
    return report

if __name__ == '__main__':
    events = get_terraform_changes(days=7)
    report = generate_report(events)
    print(json.dumps(report, indent=2, default=str))
```

### Kết Quả Báo Cáo Mẫu

```json
{
  "period": "2026-05-05 → 2026-05-12",
  "total_changes": 47,
  "changes_by_day": {
    "2026-05-05": 3,
    "2026-05-07": 12,
    "2026-05-09": 8,
    "2026-05-12": 24
  },
  "changes_by_resource_type": {
    "RunInstances": 5,
    "ModifyDBParameterGroup": 2,
    "AuthorizeSecurityGroupIngress": 1
  },
  "significant_changes": [
    {
      "time": "2026-05-09T14:30:00",
      "action": "AuthorizeSecurityGroupIngress",
      "user": "terraform-ci-role"
    }
  ]
}
```

---

## 🔒 Compliance Use Cases — Trường Hợp Tuân Thủ

### SOC 2 — Service Organization Control 2

```
SOC 2 yêu cầu:
  CC6.1: Logical access controls
  → Audit trail cho mọi thay đổi access control
  → Terraform audit trail + CloudTrail đáp ứng được

  CC7.2: System monitoring
  → Alert cho unauthorized changes
  → Drift detection đáp ứng được
```

### PCI DSS — Payment Card Industry Data Security Standard

```
PCI DSS 10.2: Implement audit trails
  → Log tất cả changes đến cardholder data environment
  → Đặc biệt quan trọng cho Terraform managing network segmentation
  → CloudTrail + SIEM đáp ứng được
```

### Cách Cấu Hình Audit Log Retention — Thời Gian Lưu Trữ

```hcl
# S3 bucket cho CloudTrail logs với lifecycle policy
resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    id     = "audit-log-retention"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"  # Chuyển sang Infrequent Access sau 30 ngày
    }

    transition {
      days          = 90
      storage_class = "GLACIER"  # Chuyển sang Glacier sau 90 ngày (rẻ hơn nhiều)
    }

    expiration {
      days = 2555  # 7 năm (yêu cầu của nhiều compliance frameworks)
    }
  }
}
```

---

## 🚫 Anti-Patterns Thường Gặp

### 1. Không Bật CloudTrail Ở Tất Cả Regions

```hcl
# ❌ Sai: CloudTrail chỉ ở một region
resource "aws_cloudtrail" "main" {
  is_multi_region_trail = false  # Nguy hiểm!
}

# ✅ Đúng: Multi-region trail
resource "aws_cloudtrail" "main" {
  is_multi_region_trail         = true
  include_global_service_events = true
}
```

### 2. Không Bảo Vệ CloudTrail Logs

```hcl
# ❌ Sai: Không có MFA delete, logs có thể bị xóa
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "my-cloudtrail-logs"
}

# ✅ Đúng: Bật versioning và log file validation
resource "aws_cloudtrail" "main" {
  enable_log_file_validation = true  # Detect nếu logs bị tamper
}

resource "aws_s3_bucket_versioning" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id
  versioning_configuration {
    status = "Enabled"
    mfa_delete = "Enabled"  # Cần MFA để xóa
  }
}
```

### 3. Không Review Audit Logs Thường Xuyên

```
❌ Sai: Có logs nhưng không bao giờ đọc
✅ Đúng: Weekly review báo cáo thay đổi
         Alert cho anomalies — bất thường
         Post-incident: luôn kiểm tra audit trail trước
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Khi có incident, bạn làm gì để tìm nguyên nhân thay đổi hạ tầng?**

> Quy trình của tôi: (1) Kiểm tra Git log và CI/CD pipeline để xem có apply nào gần đây không; (2) Query CloudTrail trong khoảng thời gian trước sự cố để tìm Write events; (3) So sánh Terraform state versions để xác định chính xác resource nào thay đổi; (4) Check Slack/Jira để xem có ai report thay đổi thủ công không. Mục tiêu là xác định root cause trong dưới 10 phút.

**Q: Làm thế nào để chứng minh compliance cho PCI DSS về audit logging?**

> Tôi dùng AWS CloudTrail với multi-region trail, enable log file validation để chứng minh logs không bị tamper, lưu vào S3 với bucket policy chỉ cho phép write (không delete), và retain ít nhất 1 năm online và 6 năm trong Glacier. Ngoài ra, kết hợp với Terraform audit trail để chứng minh mọi infrastructure change đều đi qua peer-reviewed PR process.

---

## 🔗 Tài Liệu Tham Khảo

- [AWS CloudTrail User Guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [Terraform Cloud Audit Logs API](https://developer.hashicorp.com/terraform/cloud-docs/api-docs/audit-trails)
- [GCP Cloud Audit Logs](https://cloud.google.com/logging/docs/audit)

---

**Tiếp Theo:** [4-resource-tagging.md](./4-resource-tagging.md) — Chiến lược gán nhãn tài nguyên

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
