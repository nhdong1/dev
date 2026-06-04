# Resource Tagging — Chiến Lược Gán Nhãn Tài Nguyên

> Resource Tagging — Gán Nhãn Tài Nguyên — là thực hành đặt metadata (key-value pairs — cặp khóa-giá trị) lên cloud resources để phân loại, theo dõi chi phí, quản trị, và tự động hóa. Một chiến lược tagging tốt là nền tảng cho mọi hoạt động vận hành hạ tầng quy mô lớn.

---

## 💡 Tại Sao Tagging Quan Trọng?

### Không Có Tagging Strategy

```
Cuối tháng nhận hóa đơn AWS: $45,000

Câu hỏi không trả lời được:
  ❓ Team nào phát sinh chi phí nhiều nhất?
  ❓ EC2 instance "i-0abc123456" này để làm gì?
  ❓ Database này thuộc project nào?
  ❓ Resource này có cần thiết không hay là "zombie resource"?
  
Hậu quả: Không thể tối ưu chi phí, không thể phân bổ cho đúng team
```

### Với Tagging Strategy Tốt

```
AWS Cost Explorer filter theo tags:
  Environment=production → $32,000
  Environment=staging    → $8,000
  Environment=dev        → $5,000

  Team=backend   → $20,000
  Team=data      → $15,000
  Team=platform  → $10,000

  Project=payment-service  → $18,000
  Project=user-service     → $12,000

Biết chính xác ai đang tiêu gì và tại sao
```

---

## 🏷️ Các Loại Tags Cốt Lõi

### Mandatory Tags — Tags Bắt Buộc

Mọi resource phải có những tags này:

| Tag Key | Ví Dụ Value | Mục Đích |
|---------|-------------|----------|
| `Environment` | `production`, `staging`, `dev` | Phân tách môi trường |
| `ManagedBy` | `terraform`, `manual`, `cloudformation` | Biết cách resource được tạo |
| `Owner` | `team-backend`, `john.doe@company.com` | Liên hệ khi cần |
| `Project` | `payment-service`, `user-auth` | Phân bổ chi phí theo project |
| `CostCenter` | `cc-engineering`, `cc-product` | Phân bổ chi phí theo bộ phận |

### Recommended Tags — Tags Khuyến Nghị

| Tag Key | Ví Dụ Value | Mục Đích |
|---------|-------------|----------|
| `Application` | `checkout-api`, `user-service` | Service/app cụ thể |
| `Version` | `v2.3.1`, `feature-x` | Phiên bản deploy |
| `Compliance` | `pci`, `hipaa`, `gdpr` | Quy định tuân thủ |
| `DataClassification` | `public`, `internal`, `confidential` | Phân loại dữ liệu |
| `AutoShutdown` | `true`, `false` | Tự động tắt resource ngoài giờ |
| `BackupPolicy` | `daily-7d`, `weekly-30d`, `none` | Chính sách backup |
| `GitRepo` | `github.com/company/infra` | Nguồn code |

### Terraform-Specific Tags — Tags Đặc Thù Terraform

| Tag Key | Ví Dụ Value | Mục Đích |
|---------|-------------|----------|
| `TerraformWorkspace` | `prod`, `staging` | Terraform workspace |
| `TerraformModule` | `vpc`, `eks-cluster` | Module tạo resource |
| `GitCommit` | `abc123def456` | Commit hash lúc apply |
| `LastAppliedBy` | `github-actions/john.doe` | Ai apply lần cuối |
| `LastAppliedAt` | `2026-05-12T10:30:00Z` | Khi nào apply lần cuối |

---

## 🛠️ Triển Khai Tagging Trong Terraform

### Phương Pháp 1 — `default_tags` Trong Provider (Khuyến Nghị)

Kể từ AWS provider v3.38.0, bạn có thể đặt tags mặc định ở cấp provider — tự động áp dụng cho **tất cả** resources:

```hcl
# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      # Mandatory tags — bắt buộc
      Environment = var.environment
      ManagedBy   = "terraform"
      Owner       = var.team_name
      Project     = var.project_name
      CostCenter  = var.cost_center

      # Terraform metadata
      TerraformWorkspace = terraform.workspace
      GitRepo            = var.git_repo_url
    }
  }
}

# Chú ý: default_tags sẽ được merge với tags trên từng resource
# Resource-level tags có thể override default_tags nếu cùng key
```

**Ưu điểm của `default_tags`:**
- Không cần lặp lại tags trên mỗi resource
- Đảm bảo nhất quán
- Một chỗ để thay đổi

**Nhược điểm:**
- Chỉ hoạt động với AWS provider
- Không phải mọi resource đều hỗ trợ tags (ví dụ: IAM policy)

### Phương Pháp 2 — Locals Với Merge

```hcl
# locals.tf
locals {
  # Base tags áp dụng cho mọi thứ
  base_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.team_name
    Project     = var.project_name
    CostCenter  = var.cost_center
  }
  
  # Tags cho từng layer
  network_tags = merge(local.base_tags, {
    Layer     = "network"
    Component = "vpc"
  })
  
  database_tags = merge(local.base_tags, {
    Layer            = "data"
    DataClassification = "confidential"
    BackupPolicy     = "daily-7d"
    Compliance       = "gdpr"
  })
  
  compute_tags = merge(local.base_tags, {
    Layer      = "compute"
    AutoShutdown = var.environment != "production" ? "true" : "false"
  })
}
```

**Sử dụng:**

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(local.network_tags, {
    Name = "${var.project_name}-${var.environment}-vpc"
  })
}

resource "aws_db_instance" "postgres" {
  # ... config ...
  
  tags = merge(local.database_tags, {
    Name    = "${var.project_name}-${var.environment}-postgres"
    Version = "14.9"
  })
}
```

### Phương Pháp 3 — Tagging Module (Cho Tổ Chức Lớn)

```hcl
# modules/tagging/main.tf
variable "required_tags" {
  description = "Required tags that must be provided"
  type = object({
    environment  = string
    owner        = string
    project      = string
    cost_center  = string
  })
}

variable "optional_tags" {
  description = "Optional additional tags"
  type        = map(string)
  default     = {}
}

locals {
  validated_tags = {
    Environment = var.required_tags.environment
    Owner       = var.required_tags.owner
    Project     = var.required_tags.project
    CostCenter  = var.required_tags.cost_center
    ManagedBy   = "terraform"
  }
}

output "tags" {
  description = "Merged and validated tags"
  value       = merge(local.validated_tags, var.optional_tags)
}
```

**Sử dụng tagging module:**

```hcl
module "tags" {
  source = "../../modules/tagging"
  
  required_tags = {
    environment  = "production"
    owner        = "team-backend"
    project      = "payment-service"
    cost_center  = "cc-engineering"
  }
  
  optional_tags = {
    DataClassification = "pci"
    BackupPolicy       = "daily-30d"
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  
  tags = merge(module.tags.tags, {
    Name = "payment-service-prod-01"
  })
}
```

---

## ✅ Enforcement — Bắt Buộc Tuân Thủ Tagging

### AWS Config Rules

```hcl
# Tạo AWS Config rule yêu cầu tags bắt buộc
resource "aws_config_config_rule" "required_tags" {
  name        = "required-tags-check"
  description = "Checks that required tags are present on EC2 instances and RDS"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    # Tags bắt buộc và giá trị hợp lệ
    tag1Key   = "Environment"
    tag1Value = "production,staging,dev,testing"
    tag2Key   = "ManagedBy"
    tag2Value = "terraform,cloudformation,manual"
    tag3Key   = "Owner"
    # Không kiểm tra value, chỉ kiểm tra key tồn tại
  })

  scope {
    compliance_resource_types = [
      "AWS::EC2::Instance",
      "AWS::RDS::DBInstance",
      "AWS::S3::Bucket",
      "AWS::ElasticLoadBalancingV2::LoadBalancer"
    ]
  }
}
```

### OPA — Open Policy Agent — Policy

```rego
# policies/tagging.rego
package terraform.tagging

# Danh sách tags bắt buộc
required_tags = {
  "Environment",
  "ManagedBy", 
  "Owner",
  "Project",
  "CostCenter"
}

# Các giá trị hợp lệ cho Environment
valid_environments = {"production", "staging", "dev", "testing"}

# Deny nếu resource thiếu required tags
deny[msg] {
  resource := input.resource_changes[_]
  
  # Chỉ check resources có tags
  resource.type in ["aws_instance", "aws_db_instance", "aws_s3_bucket"]
  
  # Tìm tag nào bị thiếu
  required_tag := required_tags[_]
  not resource.change.after.tags[required_tag]
  
  msg := sprintf(
    "Resource '%s' thiếu required tag: '%s'",
    [resource.address, required_tag]
  )
}

# Deny nếu Environment tag không hợp lệ
deny[msg] {
  resource := input.resource_changes[_]
  resource.type in ["aws_instance", "aws_db_instance"]
  
  env := resource.change.after.tags.Environment
  not valid_environments[env]
  
  msg := sprintf(
    "Resource '%s' có Environment tag không hợp lệ: '%s'. Phải là một trong: %v",
    [resource.address, env, valid_environments]
  )
}
```

### Checkov Policy

```python
# policies/check_required_tags.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

REQUIRED_TAGS = ["Environment", "ManagedBy", "Owner", "Project", "CostCenter"]

class CheckRequiredTags(BaseResourceCheck):
    def __init__(self):
        name = "Ensure required tags are present"
        id = "CKV_CUSTOM_TAGS_001"
        supported_resources = ['aws_instance', 'aws_db_instance', 'aws_s3_bucket']
        categories = [CheckCategories.GENERAL_SECURITY]
        super().__init__(name=name, id=id,
                        categories=categories,
                        supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        tags = conf.get('tags', [{}])
        if isinstance(tags, list):
            tags = tags[0] if tags else {}
        
        missing_tags = [tag for tag in REQUIRED_TAGS if tag not in tags]
        
        if missing_tags:
            return CheckResult.FAILED
        return CheckResult.PASSED

check = CheckRequiredTags()
```

---

## 💰 Tagging Cho Cost Allocation — Phân Bổ Chi Phí

### AWS Cost Explorer Với Tags

```hcl
# Kích hoạt tags trong Cost Explorer (thực hiện một lần)
resource "aws_ce_cost_allocation_tag" "team_tag" {
  tag_key = "Owner"
  status  = "Active"
}

resource "aws_ce_cost_allocation_tag" "environment_tag" {
  tag_key = "Environment"
  status  = "Active"
}

resource "aws_ce_cost_allocation_tag" "project_tag" {
  tag_key = "Project"
  status  = "Active"
}
```

**Query Cost Explorer theo tags:**

```bash
# Chi phí theo team trong tháng này
aws ce get-cost-and-usage \
  --time-period Start=2026-05-01,End=2026-05-31 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=TAG,Key=Owner \
  --query 'ResultsByTime[0].Groups[*].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table

# Output:
# -----------------------------------------------
# | team-backend  | 18432.50                    |
# | team-data     | 14891.20                    |
# | team-platform | 9876.30                     |
# -----------------------------------------------
```

### Budget Alerts Theo Tag

```hcl
# Tạo budget alert cho từng team
resource "aws_budgets_budget" "team_budget" {
  for_each = {
    "team-backend"   = 20000
    "team-data"      = 15000
    "team-platform"  = 10000
  }
  
  name         = "budget-${each.key}"
  budget_type  = "COST"
  limit_amount = each.value
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["Owner$${each.key}"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80  # Alert khi đạt 80%
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["team-lead@company.com"]
  }
}
```

---

## 🤖 Tag Automation — Tự Động Hóa Tagging

### AWS Lambda — Tự Động Tag Resources Mới

```python
# Lambda function tự động tag resources không có required tags
import boto3
import json

REQUIRED_TAGS = {
    "ManagedBy": "terraform",
    "Environment": "unknown",  # Default value
    "Owner": "unassigned",
}

def lambda_handler(event, context):
    """
    Triggered by CloudTrail event khi tạo EC2 instance mới.
    Tự động thêm missing required tags.
    """
    
    ec2 = boto3.client('ec2')
    
    # Lấy instance ID từ CloudTrail event
    detail = event.get('detail', {})
    instance_id = detail.get('responseElements', {}) \
                        .get('instancesSet', {}) \
                        .get('items', [{}])[0] \
                        .get('instanceId')
    
    if not instance_id:
        return
    
    # Lấy tags hiện tại
    response = ec2.describe_instances(InstanceIds=[instance_id])
    existing_tags = {
        tag['Key']: tag['Value'] 
        for tag in response['Reservations'][0]['Instances'][0].get('Tags', [])
    }
    
    # Tìm tags bị thiếu và thêm vào
    missing_tags = [
        {'Key': k, 'Value': v}
        for k, v in REQUIRED_TAGS.items()
        if k not in existing_tags
    ]
    
    if missing_tags:
        ec2.create_tags(Resources=[instance_id], Tags=missing_tags)
        print(f"Added missing tags to {instance_id}: {missing_tags}")
        
        # Alert về resource chưa được tag đúng
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:ap-southeast-1:123456:infra-alerts',
            Subject=f"⚠️ Untagged Resource: {instance_id}",
            Message=f"Instance {instance_id} thiếu required tags và đã được tự động tag. "
                    f"Vui lòng cập nhật Terraform code để thêm tags đúng."
        )
```

---

## 📏 Tag Naming Conventions — Quy Ước Đặt Tên

### Chuẩn Đặt Tên Cho Tags

```
Quy tắc chung:
  ✅ PascalCase cho keys:    Environment, CostCenter, DataClassification
  ✅ lowercase cho values:   production, team-backend, pci-dss
  ❌ Không dùng spaces:      "Cost Center" → CostCenter
  ❌ Không dùng / trong key: "Team/Owner" → TeamOwner
  ❌ Không inconsistent:     env + Environment + environment → chọn một
```

**Template chuẩn:**

```hcl
# variables.tf — khai báo biến tag
variable "tags" {
  description = "Resource tags"
  type = object({
    Environment    = string
    Owner          = string
    Project        = string
    CostCenter     = string
    # Optional fields với default
    Application    = optional(string, "")
    Compliance     = optional(string, "none")
    AutoShutdown   = optional(string, "false")
  })
  
  validation {
    condition = contains(
      ["production", "staging", "dev", "testing"],
      var.tags.Environment
    )
    error_message = "Environment phải là một trong: production, staging, dev, testing"
  }
}
```

---

## 🚫 Anti-Patterns Về Tagging

### 1. Quá Nhiều Tags

```hcl
# ❌ Sai: 30+ tags trên mỗi resource — quá phức tạp
tags = {
  Environment      = "prod"
  Region           = "ap-southeast-1"
  AZ               = "ap-southeast-1a"
  VPC              = "vpc-0123456"
  Subnet           = "subnet-0abc123"
  # ... 25 tags nữa
}

# ✅ Đúng: 5-10 tags thiết yếu
# Region, AZ, VPC, Subnet là thông tin tự có trong ARN/metadata
```

### 2. Tags Không Nhất Quán

```hcl
# ❌ Sai: Cùng khái niệm nhưng nhiều cách viết
# resource A:
tags = { env = "prod" }
# resource B:
tags = { environment = "production" }
# resource C:
tags = { Environment = "Production" }

# ✅ Đúng: Chuẩn hóa một lần, dùng locals/default_tags
tags = { Environment = "production" }  # Duy nhất một cách
```

### 3. Hardcode Tags Trong Từng Resource

```hcl
# ❌ Sai: Copy-paste tags khắp nơi
resource "aws_instance" "web" {
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    Owner       = "team-backend"
    # Phải sửa ở mỗi resource khi owner thay đổi
  }
}

# ✅ Đúng: Dùng locals hoặc default_tags
resource "aws_instance" "web" {
  tags = local.base_tags  # Thay đổi một chỗ
}
```

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Bạn thiết kế tagging strategy cho tổ chức với 5 teams và 3 môi trường như thế nào?**

> Tôi bắt đầu với mandatory tags: Environment, Owner, Project, CostCenter, ManagedBy. Implement qua `default_tags` trong AWS provider để tất cả resources tự động có base tags. Dùng AWS Config rules hoặc OPA policies để enforce và alert khi thiếu. Enable cost allocation tags trong Cost Explorer để mỗi team có thể xem chi phí của mình. Cuối cùng, tạo budget alerts per team để chủ động kiểm soát chi phí.

**Q: Làm thế nào để migrate untagged resources sang tagging strategy mới?**

> Đây là challenge thực tế. Tôi dùng approach phased: (1) Tạo Lambda function để automatically tag new resources; (2) Dùng AWS Resource Groups Tagging API để audit và bulk-tag existing resources; (3) Dùng `terraform import` để bring unmanaged resources vào Terraform cùng với proper tags; (4) Đặt deadline cho teams để clean up. Không cố gắng migrate tất cả cùng lúc — tạo technical debt nếu vội vàng.

---

## 🔗 Tài Liệu Tham Khảo

- [AWS Tagging Best Practices Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tagging-best-practices.html)
- [Terraform AWS Provider default_tags](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#default_tags-configuration-block)
- [AWS Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)

---

**Tiếp Theo:** [5-alerting.md](./5-alerting.md) — Cảnh báo khi có thay đổi ngoài dự kiến

---

**Cập Nhật Lần Cuối:** 2026-05-12
**Phiên Bản:** 1.0
