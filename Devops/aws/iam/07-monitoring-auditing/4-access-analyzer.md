# IAM Access Analyzer — Phát Hiện External Access

> **IAM Access Analyzer** (Bộ Phân Tích Truy Cập IAM) tự động phân tích toàn bộ resource policies để tìm những tài nguyên có thể truy cập từ bên ngoài tổ chức — trước khi attacker tìm thấy chúng.

---

## 🎯 Access Analyzer Là Gì?

**IAM Access Analyzer** dùng **automated reasoning** (suy luận tự động dựa trên toán học) để phân tích resource policies và xác định liệu tài nguyên có thể truy cập từ entity ngoài **zone of trust** (vùng tin tưởng) không.

### Zone Of Trust — Vùng Tin Tưởng

Khi tạo Analyzer, bạn định nghĩa zone of trust:

| Loại Analyzer | Zone of Trust | Phát hiện external access từ |
|---|---|---|
| **Account** | Một AWS account | Ngoài account đó |
| **Organization** | Toàn bộ AWS Organization | Ngoài Organization |

```
Account Analyzer:
  Tài nguyên trong Account A → Trust: Account A
  → S3 bucket share với Account B = FINDING ⚠️

Organization Analyzer:
  Tài nguyên trong Org → Trust: Toàn bộ Org
  → S3 bucket share với Account ngoài Org = FINDING ⚠️
  → S3 bucket share giữa các accounts trong Org = OK ✅
```

---

## 🔍 Những Gì Access Analyzer Phân Tích

### Các Loại Tài Nguyên Được Hỗ Trợ

| Tài nguyên | Policy được kiểm tra |
|---|---|
| **S3 Buckets** | Bucket policy + ACL |
| **IAM Roles** | Trust policy |
| **KMS Keys** | Key policy |
| **Lambda Functions** | Resource-based policy |
| **Lambda Layers** | Layer permission policy |
| **SQS Queues** | Queue policy |
| **SNS Topics** | Topic policy |
| **Secrets Manager Secrets** | Resource policy |
| **AWS Glue** | Resource policies |
| **Amazon EFS** | File system policy |

### Ví Dụ Finding — Phát Hiện Access Ngoài Ý Muốn

**S3 Bucket bị public:**
```json
{
  "findingId": "a1b2c3d4-...",
  "resourceType": "AWS::S3::Bucket",
  "resource": "arn:aws:s3:::my-prod-bucket",
  "status": "ACTIVE",
  "condition": {},
  "principal": {
    "AWS": "*"           // Mọi người đều có thể truy cập!
  },
  "action": ["s3:GetObject"],
  "isPublic": true,
  "error": null,
  "createdAt": "2026-05-16T08:00:00Z"
}
```

**IAM Role với trust policy mở quá rộng:**
```json
{
  "resourceType": "AWS::IAM::Role",
  "resource": "arn:aws:iam::123456789012:role/DataProcessingRole",
  "principal": {
    "AWS": "arn:aws:iam::999999999999:root"  // External account!
  },
  "action": ["sts:AssumeRole"],
  "isPublic": false
}
```

---

## ⚙️ Thiết Lập Access Analyzer

### Tạo Analyzer Qua CLI

```bash
# Tạo Account-level Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name "account-security-analyzer" \
  --type ACCOUNT \
  --tags '{"Environment": "production", "Purpose": "security"}'

# Tạo Organization-level Analyzer (từ management account)
aws accessanalyzer create-analyzer \
  --analyzer-name "org-security-analyzer" \
  --type ORGANIZATION

# Xem danh sách analyzers
aws accessanalyzer list-analyzers \
  --query 'analyzers[*].{Name:name,Type:type,Status:status}'
```

### Terraform Setup

```hcl
# Account Analyzer
resource "aws_accessanalyzer_analyzer" "account" {
  analyzer_name = "account-security-analyzer"
  type          = "ACCOUNT"

  tags = {
    Environment = "production"
    Purpose     = "security-monitoring"
  }
}

# Organization Analyzer (deploy ở management account)
resource "aws_accessanalyzer_analyzer" "organization" {
  provider      = aws.management_account
  analyzer_name = "org-security-analyzer"
  type          = "ORGANIZATION"
}
```

---

## 📋 Quản Lý Findings

### Xem Findings

```bash
# Liệt kê tất cả active findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:ap-southeast-1:123456789012:analyzer/account-security-analyzer \
  --filter '{"status": {"eq": ["ACTIVE"]}}' \
  --query 'findings[*].{Id:id,Resource:resource,Type:resourceType,Public:isPublic}'

# Xem findings theo loại tài nguyên
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:accessanalyzer:ap-southeast-1:123456789012:analyzer/account-security-analyzer \
  --filter '{"resourceType": {"eq": ["AWS::S3::Bucket"]}, "isPublic": {"eq": ["true"]}}'

# Xem chi tiết một finding
aws accessanalyzer get-finding \
  --analyzer-arn arn:aws:accessanalyzer:ap-southeast-1:123456789012:analyzer/account-security-analyzer \
  --id "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

### Trạng Thái Finding Và Lifecycle

```
Finding mới xuất hiện → ACTIVE
         │
         ├─ Nếu access bị thu hồi → RESOLVED (tự động)
         │
         └─ Nếu bạn xem xét và chấp nhận là hợp lệ → ARCHIVED (thủ công)
                    │
                    └─ Nếu policy thay đổi tạo ra finding mới → ACTIVE (lại)
```

### Archive Finding (Chấp Nhận Là Hợp Lệ)

```bash
# Archive một finding cụ thể (đã review và xác nhận hợp lệ)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:accessanalyzer:ap-southeast-1:123456789012:analyzer/account-security-analyzer \
  --ids '["a1b2c3d4-e5f6-7890-abcd-ef1234567890"]' \
  --status ARCHIVED
```

### Archive Rules — Tự Động Archive Findings Hợp Lệ

**Archive Rules** (Quy Tắc Lưu Trữ) tự động archive các findings phù hợp với tiêu chí đã định nghĩa — tránh phải manual review những trường hợp đã biết là hợp lệ.

```bash
# Tự động archive findings cho cross-account access từ account partner đã biết
aws accessanalyzer create-archive-rule \
  --analyzer-name account-security-analyzer \
  --rule-name "approved-cross-account-access" \
  --filter '{
    "principal.AWS": {
      "contains": ["arn:aws:iam::987654321098:root"]
    },
    "resourceType": {
      "eq": ["AWS::S3::Bucket"]
    }
  }'

# Archive tất cả findings liên quan đến CloudFront OAI (Origin Access Identity)
aws accessanalyzer create-archive-rule \
  --analyzer-name account-security-analyzer \
  --rule-name "cloudfront-oai-access" \
  --filter '{
    "principal.Service": {
      "contains": ["cloudfront.amazonaws.com"]
    }
  }'
```

---

## 🔔 Tích Hợp Với EventBridge Để Alert Realtime

```json
// EventBridge rule: phát hiện finding mới từ Access Analyzer
{
  "source": ["aws.access-analyzer"],
  "detail-type": ["Access Analyzer Finding"],
  "detail": {
    "status": ["ACTIVE"]
  }
}
```

```python
# Lambda handler xử lý Access Analyzer finding
import json
import boto3

def lambda_handler(event, context):
    finding = event['detail']
    
    resource_type = finding['resourceType']
    resource_arn = finding['resource']
    is_public = finding.get('isPublic', False)
    principal = finding.get('principal', {})
    
    # Tạo message cảnh báo
    if is_public:
        severity = "🔴 CRITICAL"
        message = f"Tài nguyên PUBLIC bị phát hiện: {resource_arn}"
    else:
        severity = "🟡 WARNING"
        external_principal = json.dumps(principal)
        message = f"Cross-account access: {resource_arn} chia sẻ với {external_principal}"
    
    # Gửi Slack notification
    slack_message = {
        "text": f"*Access Analyzer Alert {severity}*",
        "attachments": [{
            "color": "danger" if is_public else "warning",
            "fields": [
                {"title": "Resource", "value": resource_arn, "short": False},
                {"title": "Type", "value": resource_type, "short": True},
                {"title": "Public", "value": str(is_public), "short": True},
                {"title": "Principal", "value": json.dumps(principal), "short": False}
            ]
        }]
    }
    
    # Gửi đến SNS
    boto3.client('sns').publish(
        TopicArn='arn:aws:sns:ap-southeast-1:123456789012:SecurityAlerts',
        Message=json.dumps(slack_message),
        Subject=f"Access Analyzer: {severity} - {resource_type}"
    )
    
    return {"statusCode": 200}
```

---

## 🛡️ Access Analyzer Policy Validation — Xác Thực Policy

**Policy Validation** (Xác Thực Policy) là tính năng của Access Analyzer kiểm tra IAM policies trước khi deploy để phát hiện lỗi cú pháp, security warnings, và suggestions.

### Validate Policy Qua CLI

```bash
# Validate một IAM identity policy
aws accessanalyzer validate-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }]
  }' \
  --policy-type IDENTITY_POLICY

# Kết quả:
# {
#   "findings": [{
#     "findingType": "SECURITY_WARNING",
#     "issueCode": "PASS_ROLE_WITH_STAR_IN_RESOURCE",
#     "learnMoreLink": "...",
#     "locations": [...],
#     "findingDetails": "Granting 'iam:PassRole' with '*' resource allows ...",
#     "findingType": "SECURITY_WARNING"
#   }]
# }
```

### Integrate Vào CI/CD Pipeline

```yaml
# GitHub Actions: validate IAM policies trước khi merge
name: Validate IAM Policies

on:
  pull_request:
    paths:
      - 'iam/**/*.json'
      - 'terraform/**/*.tf'

jobs:
  validate-policies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1

      - name: Validate IAM Policies
        run: |
          for policy_file in iam/**/*.json; do
            echo "Validating: $policy_file"
            result=$(aws accessanalyzer validate-policy \
              --policy-document "file://$policy_file" \
              --policy-type IDENTITY_POLICY \
              --query 'findings[?findingType==`SECURITY_WARNING` || findingType==`ERROR`]')
            
            if [ "$(echo $result | jq length)" -gt "0" ]; then
              echo "❌ Policy validation failed for $policy_file:"
              echo $result | jq .
              exit 1
            else
              echo "✅ $policy_file passed validation"
            fi
          done
```

---

## 🔑 Access Analyzer Unused Access — Phân Tích Quyền Không Dùng

**Unused Access Analyzer** (phát hành 2023) giúp phát hiện quyền được cấp nhưng không được dùng trong 90 ngày — vi phạm Least Privilege Principle (Nguyên Tắc Đặc Quyền Tối Thiểu).

### Bật Unused Access Analysis

```bash
# Tạo analyzer loại ACCOUNT_UNUSED_ACCESS
aws accessanalyzer create-analyzer \
  --analyzer-name "unused-access-analyzer" \
  --type ACCOUNT_UNUSED_ACCESS \
  --configuration '{
    "unusedAccess": {
      "unusedAccessAge": 90
    }
  }'
```

### Các Loại Unused Access Findings

| Finding Type | Mô tả | Hành động đề xuất |
|---|---|---|
| **UnusedPermission** | IAM action được cấp nhưng chưa dùng 90 ngày | Xóa permission không cần thiết |
| **UnusedIAMRole** | IAM Role không được assume trong 90 ngày | Xóa role hoặc xác nhận vẫn cần |
| **UnusedIAMUserPassword** | IAM user password không được dùng 90 ngày | Xóa user hoặc enforce password rotation |
| **UnusedIAMUserAccessKey** | Access key không được dùng 90 ngày | Xóa key không dùng |

### Xem Unused Access Findings

```bash
# Xem tất cả unused access findings
aws accessanalyzer list-findings-v2 \
  --analyzer-arn arn:aws:accessanalyzer:ap-southeast-1:123456789012:analyzer/unused-access-analyzer \
  --filter '{"findingType": {"eq": ["UnusedPermission"]}}' \
  --query 'findings[*].{
    Id: id,
    Principal: principal,
    Action: action,
    LastAccessed: resource
  }'
```

---

## 📊 So Sánh Các Loại Analyzer

| Tính năng | Account Analyzer | Organization Analyzer | Unused Access Analyzer |
|---|---|---|---|
| **Phát hiện** | External access từ ngoài account | External access từ ngoài Org | Quyền không được dùng |
| **Scope** | 1 account | Toàn Org | 1 account |
| **Deploy ở** | Bất kỳ account | Management account | Bất kỳ account |
| **Chi phí** | Miễn phí | Miễn phí | $0.20/IAM role/month |
| **Use case** | Kiểm tra sharing | Multi-account governance | Right-sizing permissions |

---

## 🚨 Tình Huống Thực Tế: S3 Data Breach Prevention

**Kịch bản:** Developer vô tình thêm `"Principal": "*"` vào S3 bucket policy.

**Luồng phát hiện và response:**

```
Developer commit → Terraform apply → S3 policy thay đổi
                                            │
                                            ▼
                                   Access Analyzer phân tích
                                   policy mới (trong vòng vài phút)
                                            │
                                            ▼
                                   Finding ACTIVE: S3 bucket PUBLIC
                                            │
                                            ▼
                                   EventBridge rule kích hoạt
                                            │
                                   ┌────────┴────────┐
                                   ▼                 ▼
                             Lambda gửi         Lambda tự động
                             Slack alert        revert S3 policy
                                   │                 │
                                   ▼                 ▼
                             Team nhận         Finding RESOLVED
                             cảnh báo          (tự động)
                             < 5 phút
```

**Lambda auto-remediation:**

```python
def lambda_handler(event, context):
    finding = event['detail']
    
    if finding['resourceType'] != 'AWS::S3::Bucket':
        return
    
    if not finding.get('isPublic'):
        return
    
    bucket_name = finding['resource'].split(':::')[1]
    
    # Tắt public access ngay lập tức
    s3 = boto3.client('s3')
    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )
    
    # Log sự kiện để audit
    boto3.client('logs').put_log_events(
        logGroupName='/security/auto-remediation',
        logStreamName='access-analyzer',
        logEvents=[{
            'timestamp': int(time.time() * 1000),
            'message': json.dumps({
                'action': 'blocked_public_s3',
                'bucket': bucket_name,
                'finding_id': finding['id'],
                'automated': True
            })
        }]
    )
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Sự khác biệt giữa Account Analyzer và Organization Analyzer?**

> Account Analyzer coi account của bạn là zone of trust — nó phát hiện khi tài nguyên chia sẻ với bất kỳ entity nào ngoài account đó, kể cả accounts khác trong cùng Organization. Organization Analyzer coi toàn bộ Organization là zone of trust — nó chỉ cảnh báo khi tài nguyên chia sẻ với bên ngoài Org. Tốt nhất là deploy cả hai: Organization Analyzer ở management account để phát hiện external exposure, Account Analyzer ở từng account để phát hiện unintended cross-account sharing nội bộ.

**Q: Access Analyzer có thể thay thế manual policy review không?**

> Access Analyzer bổ sung chứ không thay thế hoàn toàn manual review. Nó dùng automated reasoning — đảm bảo không bỏ sót edge cases về mặt logic — nhưng không đánh giá được context nghiệp vụ (ví dụ: tại sao một cross-account access lại nguy hiểm trong bối cảnh cụ thể). Kết hợp tối ưu: Access Analyzer tự động phát hiện findings + security engineer review để đánh giá risk và business context.

**Q: Làm thế nào tích hợp Access Analyzer vào DevSecOps pipeline?**

> Dùng `aws accessanalyzer validate-policy` trong CI/CD để block merge request nếu policy có SECURITY_WARNING hoặc ERROR findings. Dùng Archive Rules để auto-suppress findings đã được review và approved (tránh alert fatigue). Dùng EventBridge + Lambda để auto-remediate public access findings trong môi trường production. Tạo báo cáo hàng tuần từ findings để track security posture trends.

---

**Tiếp Theo:** [5-security-dashboard.md](5-security-dashboard.md) — CloudWatch Security Dashboard

---

**Cập Nhật Lần Cuối:** 2026-05-16
