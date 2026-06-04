# 5 — Audit Logging — Nhật Ký Kiểm Tra Thay Đổi Hạ Tầng

> Câu hỏi quan trọng nhất sau một sự cố: **Ai** đã thay đổi **cái gì**, vào **lúc nào**?

---

## 🎯 Tại Sao Cần Audit Logging?

### Kịch Bản Thực Tế

```
Thứ Hai 9:00 sáng: Production database bỗng không thể kết nối được
Thứ Hai 9:15 sáng: Team phát hiện security group RDS bị xóa rule inbound
Câu hỏi: Ai xóa? Khi nào? Cố ý hay do Terraform apply nhầm?
```

Không có audit log → mất hàng giờ điều tra, không có bằng chứng, không học được gì.

Với audit log đầy đủ → tìm ra ngay trong 5 phút.

---

## ☁️ AWS CloudTrail — Nhật Ký API Calls AWS

CloudTrail ghi lại **mọi** API call đến AWS, bao gồm tất cả actions Terraform thực hiện.

### Bật CloudTrail Bằng Terraform

```hcl
# cloudtrail.tf

# S3 bucket chứa logs
resource "aws_s3_bucket" "cloudtrail_logs" {
  bucket        = "my-cloudtrail-logs-${data.aws_caller_identity.current.account_id}"
  force_destroy = false  # Không cho xóa bucket nếu còn logs
}

# Bật versioning để không mất log
resource "aws_s3_bucket_versioning" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Mã hoá logs
resource "aws_s3_bucket_server_side_encryption_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail_logs.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.cloudtrail.arn
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "cloudtrail" {
  bucket                  = aws_s3_bucket.cloudtrail_logs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# S3 bucket policy cho CloudTrail
resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail_logs.id
  policy = data.aws_iam_policy_document.cloudtrail_bucket.json
}

data "aws_iam_policy_document" "cloudtrail_bucket" {
  statement {
    sid    = "AWSCloudTrailAclCheck"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }

    actions   = ["s3:GetBucketAcl"]
    resources = [aws_s3_bucket.cloudtrail_logs.arn]
  }

  statement {
    sid    = "AWSCloudTrailWrite"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }

    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.cloudtrail_logs.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"]

    condition {
      test     = "StringEquals"
      variable = "s3:x-amz-acl"
      values   = ["bucket-owner-full-control"]
    }
  }
}

# Tạo CloudTrail
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  s3_key_prefix                 = "cloudtrail"
  include_global_service_events = true  # IAM, STS events
  is_multi_region_trail         = true  # Tất cả regions
  enable_log_file_validation    = true  # Detect log tampering — phát hiện giả mạo log

  # Mã hoá với KMS
  kms_key_id = aws_kms_key.cloudtrail.arn

  # Ghi lại data events — thao tác với S3 objects và Lambda
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::my-critical-bucket/"]
    }
  }

  # CloudWatch integration để alert real-time
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  tags = {
    Name        = "main-trail"
    Environment = "production"
  }
}

# CloudWatch Log Group cho CloudTrail
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/main"
  retention_in_days = 90  # Giữ 90 ngày
  kms_key_id        = aws_kms_key.cloudtrail.arn
}
```

### Tìm Kiếm Trong CloudTrail

```bash
# Tìm ai đã xóa security group rule
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RevokeSecurityGroupIngress \
  --start-time 2026-05-10T00:00:00Z \
  --end-time 2026-05-12T23:59:59Z \
  --query 'Events[*].{Time:EventTime,User:Username,Event:CloudTrailEvent}' \
  --output table

# Tìm mọi actions của một IAM role
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=terraform-ci-prod-role \
  --start-time 2026-05-11T00:00:00Z \
  --output json | jq '.Events[].EventName' | sort | uniq -c | sort -rn

# Tìm ai tạo/xóa resources trong 24 giờ qua
aws cloudtrail lookup-events \
  --start-time $(date -d '24 hours ago' -u +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[?contains(`["CreateDBInstance","DeleteDBInstance","CreateBucket","DeleteBucket"]`, EventName)]' \
  --output table
```

---

## 🏗️ CloudWatch Alerts — Cảnh Báo Real-time

```hcl
# Cảnh báo khi có thay đổi security group
resource "aws_cloudwatch_metric_alarm" "security_group_changes" {
  alarm_name          = "SecurityGroupChanges"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "SecurityGroupEventCount"
  namespace           = "CloudTrailMetrics"
  period              = "300"
  statistic           = "Sum"
  threshold           = "1"
  alarm_description   = "Security group có thay đổi"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

# Metric filter từ CloudTrail logs
resource "aws_cloudwatch_log_metric_filter" "security_group_changes" {
  name           = "SecurityGroupChanges"
  pattern        = "{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "SecurityGroupEventCount"
    namespace = "CloudTrailMetrics"
    value     = "1"
  }
}

# SNS Topic cho alerts
resource "aws_sns_topic" "security_alerts" {
  name              = "security-alerts"
  kms_master_key_id = "alias/aws/sns"
}

resource "aws_sns_topic_subscription" "security_alerts_email" {
  topic_arn = aws_sns_topic.security_alerts.arn
  protocol  = "email"
  endpoint  = "security-team@company.com"
}
```

---

## 📊 Terraform Cloud Audit Logging

Nếu dùng Terraform Cloud — HCP Terraform, audit logs được tự động thu thập.

```bash
# Xem audit log qua API
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  "https://app.terraform.io/api/v2/organization/audit-trail" | \
  jq '.data[] | {
    timestamp: .attributes.timestamp,
    type: .attributes.type,
    actor: .attributes.auth.description,
    resource: .attributes.resource.id
  }'
```

### Terraform Cloud Audit Events Quan Trọng

```
workspace.applied          → Ai apply workspace nào, bao giờ
run.created                → Run được tạo từ đâu (VCS/API/UI)
variable.updated           → Variable thay đổi (sensitive = masked)
workspace.settings.updated → Workspace settings thay đổi
team.created/deleted       → Team được tạo/xóa
user.invited               → User được mời vào organization
```

---

## 🔍 Terraform-specific Audit Patterns

### Theo Dõi Ai Apply Gì Qua CI/CD

```yaml
# GitHub Actions — thêm metadata vào run
- name: Terraform Apply
  env:
    TF_VAR_applied_by: "${{ github.actor }}"
    TF_VAR_git_commit: "${{ github.sha }}"
    TF_VAR_pr_number: "${{ github.event.pull_request.number }}"
  run: terraform apply -auto-approve tfplan
```

```hcl
# Gắn metadata vào resources để audit sau
resource "aws_resourcegroups_group" "deployment_metadata" {
  name = "terraform-deploy-${formatdate("YYYYMMDDhhmm", timestamp())}"

  tags = {
    AppliedBy  = var.applied_by
    GitCommit  = var.git_commit
    PRNumber   = var.pr_number
    AppliedAt  = timestamp()
    ManagedBy  = "terraform"
  }
}

# Tagging tất cả resources với deployment info
locals {
  common_tags = {
    ManagedBy    = "terraform"
    GitCommit    = var.git_commit
    Environment  = var.environment
    LastApplied  = timestamp()
  }
}
```

### S3 Object Logging — Ghi Log Thao Tác Với State File

```hcl
# Bật S3 access logging cho state bucket
resource "aws_s3_bucket_logging" "state_access_log" {
  bucket = aws_s3_bucket.terraform_state.id

  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "state-access-logs/"
}

# Bật S3 server access logging
resource "aws_s3_bucket" "access_logs" {
  bucket = "my-s3-access-logs"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "access_logs" {
  bucket = aws_s3_bucket.access_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## 🔔 Alert Rules Quan Trọng Nên Thiết Lập

```hcl
# alerts.tf

# 1. Root account usage — CRITICAL
resource "aws_cloudwatch_log_metric_filter" "root_account_usage" {
  name           = "RootAccountUsage"
  pattern        = "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "RootAccountUsageCount"
    namespace = "CloudTrailMetrics"
    value     = "1"
  }
}

# 2. IAM policy changes
resource "aws_cloudwatch_log_metric_filter" "iam_policy_changes" {
  name    = "IAMPolicyChanges"
  pattern = "{($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=SetDefaultPolicyVersion)}"

  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "IAMPolicyEventCount"
    namespace = "CloudTrailMetrics"
    value     = "1"
  }
}

# 3. Unauthorized API calls
resource "aws_cloudwatch_log_metric_filter" "unauthorized_api_calls" {
  name    = "UnauthorizedAPICalls"
  pattern = "{($.errorCode=*UnauthorizedAccess*)||($.errorCode=AccessDenied*)}"

  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "UnauthorizedAPICallCount"
    namespace = "CloudTrailMetrics"
    value     = "1"
  }
}

# Alarm cho unauthorized calls cao bất thường
resource "aws_cloudwatch_metric_alarm" "unauthorized_api_calls" {
  alarm_name          = "HighUnauthorizedAPICalls"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "UnauthorizedAPICallCount"
  namespace           = "CloudTrailMetrics"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "Quá nhiều unauthorized API calls trong 5 phút"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

---

## 📈 AWS Config — Theo Dõi Configuration History

AWS Config ghi lại lịch sử thay đổi cấu hình của resources theo thời gian.

```hcl
# Bật AWS Config
resource "aws_config_configuration_recorder" "main" {
  name     = "main-recorder"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "main-channel"
  s3_bucket_name = aws_s3_bucket.config_logs.id

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true

  depends_on = [aws_config_delivery_channel.main]
}

# Config Rule: RDS phải bật encryption
resource "aws_config_config_rule" "rds_encryption" {
  name = "rds-storage-encrypted"

  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }

  depends_on = [aws_config_configuration_recorder_status.main]
}

# Config Rule: S3 không được có public access
resource "aws_config_config_rule" "s3_public_access" {
  name = "s3-bucket-public-read-prohibited"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder_status.main]
}
```

---

## 🗂️ Lưu Trữ Và Retention — Thời Gian Giữ Log

```hcl
# Lifecycle policy cho S3 — tự động chuyển sang cheaper storage
resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail_logs" {
  bucket = aws_s3_bucket.cloudtrail_logs.id

  rule {
    id     = "audit-log-lifecycle"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"  # Infrequent Access — truy cập không thường xuyên
    }

    transition {
      days          = 90
      storage_class = "GLACIER"  # Lưu trữ lạnh, chi phí thấp nhất
    }

    expiration {
      days = 2555  # Xóa sau 7 năm (compliance requirement)
    }
  }
}
```

---

## ✅ Checklist Audit Logging

- [ ] CloudTrail bật ở tất cả regions (`is_multi_region_trail = true`)
- [ ] CloudTrail bật `enable_log_file_validation` để detect tampering
- [ ] CloudTrail logs được mã hoá với KMS
- [ ] S3 bucket chứa logs không public, versioning bật
- [ ] CloudWatch Alerts cho root account usage
- [ ] CloudWatch Alerts cho IAM policy changes
- [ ] CloudWatch Alerts cho security group changes
- [ ] Retention policy phù hợp với compliance (tối thiểu 1 năm)
- [ ] AWS Config bật và có rules cho compliance
- [ ] State bucket có S3 access logging

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Làm sao biết ai đã chạy terraform apply trong production?**
> Ba nguồn: 1) CloudTrail logs — tìm theo IAM role/user và thời gian; 2) Terraform Cloud audit trail — nếu dùng TFC; 3) CI/CD pipeline logs — GitHub Actions/GitLab CI ghi actor và commit SHA. Tốt nhất là kết hợp cả ba và dùng resource tags để link Terraform apply với CloudTrail events.

**Q: Sự khác nhau giữa CloudTrail và AWS Config?**
> CloudTrail ghi **API calls** — ai gọi API gì, khi nào, từ IP nào. AWS Config ghi **configuration state** — resource được cấu hình như thế nào tại từng thời điểm. CloudTrail trả lời "Ai làm gì?", AWS Config trả lời "Resource này trước đây cấu hình thế nào?". Dùng cả hai để có đầy đủ audit trail.

**Q: Terraform state file có cần audit log riêng không?**
> Có. Bật S3 server access logging cho state bucket để ghi lại ai đọc/ghi state, bao giờ. Kết hợp với CloudTrail data events cho S3 để có log chi tiết hơn. Điều này giúp phát hiện nếu state file bị truy cập trái phép từ bên ngoài Terraform workflow.
