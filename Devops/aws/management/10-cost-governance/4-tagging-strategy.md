# Tag Strategy & Cost Allocation — Chiến Lược Gắn Nhãn Tài Nguyên

> **Resource Tagging** (Gắn Nhãn Tài Nguyên) là nền tảng của mọi Cost Governance strategy — không có tag tốt, không thể phân bổ chi phí chính xác. Kết hợp **Tag Policies** (Chính Sách Tag) trong Organizations, **Cost Allocation Tags** (Tag Phân Bổ Chi Phí), và **AWS Config rules** để tạo hệ thống tag enforcement toàn diện.

---

## 📚 Mục Lục

1. [Tại Sao Tag Strategy Quan Trọng?](#tại-sao-tag-strategy-quan-trọng)
2. [Taxonomy Tag Chuẩn](#taxonomy-tag-chuẩn)
3. [Cost Allocation Tags](#cost-allocation-tags)
4. [Tag Policies trong AWS Organizations](#tag-policies-trong-aws-organizations)
5. [Enforcement với AWS Config](#enforcement-với-aws-config)
6. [SCP Enforce Tagging](#scp-enforce-tagging)
7. [Tag Governance Pipeline](#tag-governance-pipeline)
8. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tại Sao Tag Strategy Quan Trọng?

### Vấn Đề Không Có Tag

```
Hóa đơn AWS tháng: $45,000

Câu hỏi không trả lời được:
  ❌ Team nào tốn nhiều nhất?
  ❌ Project nào đang vượt ngân sách?
  ❌ Môi trường prod tốn bao nhiêu vs dev?
  ❌ Cái gì chạy mà không ai nhận?

→ Không thể chargeback, không thể optimize,
  không thể giải thích cho CFO
```

### Có Tag Strategy Tốt

```
Hóa đơn AWS tháng: $45,000

Trả lời được ngay:
  ✅ Team payment: $18,000 (40%)  → vượt budget $2,000
  ✅ Project apollo: $12,000 (27%) → on track
  ✅ Env prod: $35,000 | dev: $7,000 | staging: $3,000
  ✅ Untagged resources: $250     → cần điều tra ngay

→ Chargeback chính xác, optimize đúng chỗ,
  báo cáo tài chính minh bạch
```

---

## 🏷️ Taxonomy Tag Chuẩn

### Mandatory Tags (Tag Bắt Buộc)

Đây là các tag **tối thiểu** phải có trên mọi resource:

| Tag Key        | Ví Dụ Giá Trị                      | Mục Đích                          |
| -------------- | ---------------------------------- | --------------------------------- |
| `Environment`  | `prod`, `staging`, `dev`, `test`   | Phân tách môi trường              |
| `Team`         | `backend`, `frontend`, `platform`  | Gán trách nhiệm team              |
| `Project`      | `user-auth`, `payment`, `analytics`| Phân bổ chi phí theo project      |
| `CostCenter`   | `CC-1001`, `CC-2003`               | Mapping với hệ thống kế toán      |
| `Owner`        | `john.doe@company.com`             | Người chịu trách nhiệm            |

### Recommended Tags (Tag Nên Có)

| Tag Key        | Ví Dụ Giá Trị                      | Mục Đích                          |
| -------------- | ---------------------------------- | --------------------------------- |
| `Application`  | `api-gateway`, `worker-service`    | Phân biệt các app trong project   |
| `Version`      | `v2.1.0`, `latest`                 | Tracking deployment version       |
| `ManagedBy`    | `terraform`, `cloudformation`, `manual` | Quản lý IaC provenance       |
| `DataClass`    | `public`, `internal`, `confidential` | Phân loại độ nhạy dữ liệu      |
| `Compliance`   | `pci`, `hipaa`, `sox`              | Đánh dấu scope compliance         |
| `AutoShutdown` | `true`, `false`                    | Cho phép tự động tắt ngoài giờ   |

### Tag Naming Conventions (Quy Ước Đặt Tên)

```
Nguyên tắc:
  ✅ PascalCase cho keys: Environment, CostCenter
  ✅ lowercase cho values: prod, dev, backend
  ✅ Dùng hyphen, không dùng underscore trong values
  ✅ Nhất quán toàn organization

  ❌ env vs Environment vs environment (3 keys khác nhau!)
  ❌ PROD vs Prod vs prod (3 values khác nhau!)
  ❌ Giá trị tự do không controlled vocabulary

Controlled vocabulary (Bộ từ vựng kiểm soát):
  Environment:
    Allowed: [prod, staging, dev, test, sandbox]
  Team:
    Allowed: [platform, backend, frontend, data, security, devops]
```

> **Lưu ý kỹ thuật:** AWS tag keys phân biệt hoa/thường (case-sensitive). `environment` và `Environment` là hai tag **khác nhau**.

---

## 💰 Cost Allocation Tags

**Cost Allocation Tags** (Tag Phân Bổ Chi Phí) là tags được **kích hoạt** trong Billing Console để hiển thị trong Cost Explorer và CUR.

### Hai Loại Cost Allocation Tags

#### 1. AWS-generated tags (Tag do AWS tạo)

```
Ví dụ:
  aws:createdBy      → IAM user/role tạo resource
  aws:cloudformation:stack-name → Tên CloudFormation stack

Kích hoạt trong Billing Console → tự động có trong Cost Explorer
Không cần thêm thủ công vào resource
```

#### 2. User-defined tags (Tag do người dùng tạo)

```
Các tag bạn tự tạo: Environment, Team, Project, CostCenter...

Quy trình kích hoạt:
  1. Tạo tag trên resource (EC2, RDS, S3...)
  2. Billing Console → Cost allocation tags → Activate
  3. Chờ 24 giờ để xuất hiện trong Cost Explorer/CUR
  
Lưu ý quan trọng:
  → Chỉ tag trên RESOURCE mới count vào cost allocation
  → Tag trên resource TRƯỚC khi kích hoạt trong Billing
    vẫn được backfill (không mất dữ liệu lịch sử)
  → Tối đa 500 cost allocation tags per account
```

### Kích Hoạt Cost Allocation Tags

```
Billing and Cost Management Console
  → Cost allocation tags
  → AWS-generated tags tab → Activate "aws:createdBy"
  → User-defined tags tab → Activate: Environment, Team, Project, CostCenter
  
Sau khi activate (24h):
  Cost Explorer → Group by tag: Team
    → backend:  $18,500
    → frontend: $4,200
    → platform: $12,800
    → (no team tag): $450  ← cần fix
```

---

## 🏛️ Tag Policies trong AWS Organizations

**Tag Policies** (Chính Sách Tag) trong **AWS Organizations** (Tổ Chức AWS) chuẩn hóa cách tag được dùng trên toàn organization.

### Tag Policy là gì?

```
Tag Policy KHÔNG enforce (không bắt buộc tạo tag).
Tag Policy CÀI:
  ✅ Chuẩn hóa key name (case)
  ✅ Giới hạn allowed values
  ✅ Báo non-compliant khi vi phạm
  ✅ Có thể prevent non-compliant tagging (opt-in)
```

### Ví Dụ Tag Policy

```json
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["prod", "staging", "dev", "test", "sandbox"]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "ec2:volume",
          "rds:db",
          "s3:bucket",
          "lambda:function"
        ]
      }
    },
    "Team": {
      "tag_key": {
        "@@assign": "Team"
      },
      "tag_value": {
        "@@assign": [
          "platform", "backend", "frontend",
          "data", "security", "devops"
        ]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "rds:db"]
      }
    }
  }
}
```

### Attach Tag Policy lên OU/Account

```
Organizations Console
  → Policies → Tag policies → Create policy
  → Paste JSON policy
  → Attach to: OU hoặc Account

Inheritance (Kế Thừa):
  Root policy + OU policy → child accounts nhận merged policy
  Giá trị allowed = giao của tất cả policies áp dụng
```

### Kiểm Tra Compliance Tag Policy

```
Organizations Console → Tag policies → [policy name]
  → View compliance summary
  
Hoặc API:
aws organizations list-targets-for-policy \
  --policy-id p-xxxxxxxxxxxx

aws organizations describe-effective-policy \
  --policy-type TAG_POLICY \
  --target-id 123456789012
```

---

## 🔍 Enforcement với AWS Config

**AWS Config Rule** `required-tags` phát hiện resource **thiếu** mandatory tags.

### Config Rule: required-tags

```json
{
  "ConfigRuleName": "required-tags",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "REQUIRED_TAGS"
  },
  "InputParameters": {
    "tag1Key": "Environment",
    "tag2Key": "Team",
    "tag3Key": "Project",
    "tag4Key": "CostCenter",
    "tag5Key": "Owner"
  },
  "Scope": {
    "ComplianceResourceTypes": [
      "AWS::EC2::Instance",
      "AWS::RDS::DBInstance",
      "AWS::S3::Bucket",
      "AWS::Lambda::Function",
      "AWS::ElasticLoadBalancingV2::LoadBalancer"
    ]
  }
}
```

### Tự Động Remediation Với SSM

```yaml
# Auto-remediation: Gắn tag mặc định khi thiếu
RemediationConfiguration:
  ConfigRuleName: required-tags
  TargetType: SSM_DOCUMENT
  TargetId: AWS-AddTagsToResources
  Parameters:
    ResourceType:
      StaticValue:
        Values:
          - EC2
    ResourceIds:
      ResourceValue:
        Value: RESOURCE_ID
    Tags:
      StaticValue:
        Values:
          - "Owner=UNKNOWN-requires-review"
  Automatic: true
  MaximumAutomaticAttempts: 3
  RetryAttemptSeconds: 60
```

> **Lưu ý:** Gắn tag `Owner=UNKNOWN-requires-review` để tạo visibility mà không block operations. Team có thể filter và fix sau.

### Custom Config Rule (Lambda)

```python
import boto3
import json

def evaluate_compliance(configuration_item, rule_parameters):
    """Kiểm tra resource có đủ required tags không."""
    
    required_tags = ['Environment', 'Team', 'Project', 'CostCenter']
    resource_tags = configuration_item.get('tags', {})
    
    missing_tags = [
        tag for tag in required_tags
        if tag not in resource_tags or not resource_tags[tag]
    ]
    
    if missing_tags:
        return {
            'compliance_type': 'NON_COMPLIANT',
            'annotation': f"Missing required tags: {', '.join(missing_tags)}"
        }
    
    # Validate tag values
    allowed_environments = ['prod', 'staging', 'dev', 'test', 'sandbox']
    env_value = resource_tags.get('Environment', '')
    
    if env_value not in allowed_environments:
        return {
            'compliance_type': 'NON_COMPLIANT',
            'annotation': f"Invalid Environment value: '{env_value}'. "
                          f"Allowed: {allowed_environments}"
        }
    
    return {
        'compliance_type': 'COMPLIANT',
        'annotation': 'All required tags present and valid'
    }
```

---

## 🛡️ SCP Enforce Tagging

**SCP — Service Control Policy** (Chính Sách Kiểm Soát Dịch Vụ) là cách **mạnh nhất** để bắt buộc tagging — chặn tạo resource nếu không có tag.

### SCP Deny Without Tags

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEC2WithoutMandatoryTags",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Environment": "true"
        }
      }
    },
    {
      "Sid": "DenyEC2WithoutTeamTag",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Team": "true"
        }
      }
    }
  ]
}
```

### SCP Enforce Tag Values

```json
{
  "Sid": "DenyInvalidEnvironmentTag",
  "Effect": "Deny",
  "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestTag/Environment": [
        "prod", "staging", "dev", "test", "sandbox"
      ]
    }
  }
}
```

> **Cảnh báo:** SCP cứng có thể block automation tools nếu chúng không set tags. Kiểm tra IaC pipelines (Terraform, CDK, CloudFormation) đều có tags đúng trước khi apply SCP.

---

## 🔄 Tag Governance Pipeline

### Luồng Kiểm Soát Tag End-to-End

```
Developer tạo resource (EC2, RDS, S3...)
           │
           ▼
    IaC Template (Terraform/CDK/CFN)
    [bắt buộc có required_tags variable]
           │
           ▼
   CI/CD Pipeline — Tag Validation
   [check tags trước khi deploy]
           │
           ▼
    SCP Gate — Block nếu thiếu tag
           │
           ▼
    Resource Created ✅
           │
           ▼
  AWS Config — Continuous Monitoring
  [detect resource mới thiếu tag]
           │
      ┌────┴────┐
      │         │
  COMPLIANT  NON-COMPLIANT
      │         │
      ▼         ▼
  Monitor    Auto-remediation
  tiếp tục   + Alert team lead
             + Jira ticket tự động
```

### Terraform Module Enforcing Tags

```hcl
# modules/required-tags/variables.tf
variable "required_tags" {
  description = "Mandatory tags for all resources"
  type = object({
    Environment = string
    Team        = string
    Project     = string
    CostCenter  = string
    Owner       = string
  })
  
  validation {
    condition = contains(
      ["prod", "staging", "dev", "test", "sandbox"],
      var.required_tags.Environment
    )
    error_message = "Environment must be one of: prod, staging, dev, test, sandbox"
  }
}

# Usage in resource:
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = merge(var.required_tags, {
    Name = var.instance_name
  })
}
```

---

## ✅ Thực Hành Tốt Nhất

### 1. Bắt Đầu Đơn Giản, Mở Rộng Dần

```
Phase 1: 3 mandatory tags (Environment, Team, Owner)
Phase 2: Thêm Project, CostCenter
Phase 3: Thêm Application, DataClass
Phase 4: Full enforcement với SCP
```

### 2. Tag Resources Ngay Khi Tạo

```
Retroactive tagging (gắn tag sau) tốn công sức hơn nhiều
  → Hàng trăm resource không tagged = nhiều ngày công

Best practice:
  → IaC templates luôn có required tags
  → Không cho phép tạo resource thủ công trên Console
    (hoặc require review process)
```

### 3. Audit Thường Xuyên

```
Hàng tuần:
  → Cost Explorer: Group by Tag (Team) → xem "no tag" category
  → Config dashboard: Review NON_COMPLIANT resources

Hàng tháng:
  → Report untagged cost % → KPI cho mỗi team
  → Target: < 1% cost không có tag
```

### 4. Tự Động Hóa Phát Hiện Untagged Resources

```python
# Script hàng ngày: tìm EC2 thiếu required tags
import boto3

def find_untagged_instances():
    ec2 = boto3.client('ec2')
    required_tags = {'Environment', 'Team', 'Project', 'CostCenter'}
    
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    untagged = []
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            existing_keys = {t['Key'] for t in instance.get('Tags', [])}
            missing = required_tags - existing_keys
            
            if missing:
                untagged.append({
                    'InstanceId': instance['InstanceId'],
                    'MissingTags': list(missing),
                    'LaunchTime': str(instance['LaunchTime'])
                })
    
    return untagged
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Tag Policies trong Organizations có enforce mandatory tags không?**
> Mặc định, **Tag Policies không block** việc tạo resource thiếu tag — chúng chỉ **report non-compliance**. Để enforce thực sự (block tạo resource), cần dùng **SCP với điều kiện `aws:RequestTag`**. Kết hợp: Tag Policies chuẩn hóa tên/giá trị + SCP enforce mandatory tags + Config rule detect resource đã tồn tại vi phạm.

**Q: Cost Allocation Tags khác Tag thường như thế nào?**
> Tag thường chỉ là metadata trên resource. **Cost Allocation Tags** phải được **kích hoạt thủ công** trong Billing Console — sau khi kích hoạt mới xuất hiện trong Cost Explorer và CUR để phân tích chi phí. Chờ 24 giờ sau khi activate để data xuất hiện.

**Q: Làm thế nào phát hiện và fix untagged resources?**
> Ba hướng: (1) **AWS Config rule `required-tags`** tự động detect và có thể auto-remediate qua SSM. (2) **Tag Editor** (công cụ trong AWS Console) để search và tag hàng loạt. (3) **Cost Explorer** group by tag → xem category "no tag value" → drill-down xem service nào. Prevention tốt hơn cure: enforce từ IaC templates và SCP.

**Q: Đặt bao nhiêu mandatory tags là hợp lý?**
> **3–5 tags** là sweet spot: đủ để phân bổ chi phí nhưng không quá phức tạp để maintain. Tối thiểu: `Environment`, `Team`, `Owner`. Nếu cần chargeback chính xác: thêm `Project` và `CostCenter`. Quá nhiều mandatory tags → developer bỏ qua hoặc điền sai.

---

**Điều Hướng:**
← [3-anomaly-detection.md](3-anomaly-detection.md) | → [5-savings-plans-ri.md](5-savings-plans-ri.md)

**Cập Nhật Lần Cuối:** 2026-05-17
