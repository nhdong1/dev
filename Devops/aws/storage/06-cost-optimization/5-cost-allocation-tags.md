# Cost Allocation Tags — Thẻ Phân Bổ Chi Phí

> Cost Allocation Tags — Thẻ Phân Bổ Chi Phí cho phép phân tích chi phí AWS theo team, project, environment hoặc bất kỳ chiều phân loại nào. Đây là nền tảng của FinOps — Financial Operations — Vận Hành Tài Chính Đám Mây.

## 📚 Mục Lục

1. [Tổng Quan Cost Allocation Tags](#1-tổng-quan-cost-allocation-tags)
2. [Hai Loại Tags](#2-hai-loại-tags)
3. [Chiến Lược Tag Storage Resources](#3-chiến-lược-tag-storage-resources)
4. [Kích Hoạt Cost Allocation Tags](#4-kích-hoạt-cost-allocation-tags)
5. [Phân Tích Chi Phí Với Cost Explorer](#5-phân-tích-chi-phí-với-cost-explorer)
6. [Tag Enforcement — Bắt Buộc Gắn Tag](#6-tag-enforcement)
7. [Thực Hành: Tag Policy Cho Tổ Chức](#7-thực-hành-tag-policy-cho-tổ-chức)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Cost Allocation Tags

### Vấn Đề Tags Giải Quyết

```
Không có tags:
  Bill tháng = $50,000 tổng
  "Chi phí S3 $15,000" → Không biết của team nào? Project nào?
  Không thể charge back từng team
  Không thể phát hiện team nào overspend

Có Cost Allocation Tags:
  Bill tháng = $50,000 tổng
  Team Backend: $12,000 (S3: $5K, EBS: $4K, EFS: $3K)
  Team Frontend: $8,000 (S3: $7K, EBS: $1K)
  Team Data: $30,000 (S3: $20K, EBS: $10K)

  → Ngay lập tức thấy Data team chiếm 60% chi phí
  → Có thể implement chargeback (tính phí nội bộ)
  → Biết chính xác cần tối ưu ở đâu
```

### Tags Không Làm Gì

```
Tags KHÔNG:
  ❌ Tự động giảm chi phí
  ❌ Phân quyền truy cập (đó là IAM)
  ❌ Ảnh hưởng đến performance
  ❌ Tự động áp dụng cho resources mới

Tags CÓ thể:
  ✅ Phân loại chi phí để phân tích
  ✅ Là cơ sở cho budget alerts
  ✅ Hỗ trợ showback/chargeback (hiển thị/tính phí chi phí nội bộ)
  ✅ Giúp identify resources không cần thiết
  ✅ Là điều kiện trong IAM policies và SCPs
```

---

## 2. Hai Loại Tags

### AWS-Generated Tags — Tags AWS Tạo Sẵn

```
AWS tự động tạo một số tags:
  aws:createdBy → IAM user/role tạo resource
  aws:cloudformation:stack-name → CloudFormation stack tạo resource
  aws:autoscaling:groupName → Auto Scaling group

Không thể edit hoặc delete
Phải activate trong Billing Console để dùng cho cost allocation
```

### User-Defined Tags — Tags Do Người Dùng Định Nghĩa

```
Bạn tự tạo key-value pairs:
  Key: "Environment" | Value: "production", "staging", "dev"
  Key: "Team" | Value: "backend", "frontend", "data"
  Key: "Project" | Value: "payment-service", "user-portal"
  Key: "CostCenter" | Value: "CC-001", "CC-002" (mã trung tâm chi phí)
  Key: "Owner" | Value: "john@company.com"

Giới hạn:
  - Tối đa 50 user-defined tags per resource
  - Key: tối đa 128 ký tự
  - Value: tối đa 256 ký tự
  - Phân biệt hoa thường (case-sensitive): "env" ≠ "Env"
```

---

## 3. Chiến Lược Tag Storage Resources

### Framework Tag Cốt Lõi Cho Storage

```
Bộ tag tối thiểu cho mọi S3 bucket và EBS volume:

┌──────────────────┬──────────────────────────────────────────┐
│ Key              │ Value (Ví Dụ)                            │
├──────────────────┼──────────────────────────────────────────┤
│ env              │ prod / staging / dev / test              │
│ team             │ backend / frontend / data / devops       │
│ project          │ payment-service / user-auth / analytics  │
│ cost-center      │ CC-BACKEND-001 / CC-DATA-002             │
│ owner            │ john.doe@company.com                     │
│ managed-by       │ terraform / cloudformation / manual      │
└──────────────────┴──────────────────────────────────────────┘
```

### Tag Cho S3 Buckets

```bash
# Gắn tags vào S3 bucket
aws s3api put-bucket-tagging \
  --bucket my-production-assets \
  --tagging '{
    "TagSet": [
      {"Key": "env", "Value": "prod"},
      {"Key": "team", "Value": "frontend"},
      {"Key": "project", "Value": "user-portal"},
      {"Key": "cost-center", "Value": "CC-FE-001"},
      {"Key": "data-classification", "Value": "public"},
      {"Key": "managed-by", "Value": "terraform"}
    ]
  }'

# Xem tags của bucket
aws s3api get-bucket-tagging --bucket my-production-assets
```

### Tag Cho EBS Volumes

```bash
# Gắn tags vào EBS volume
aws ec2 create-tags \
  --resources vol-xxxxxxxxxxxxxxxxx \
  --tags \
    Key=env,Value=prod \
    Key=team,Value=backend \
    Key=project,Value=payment-service \
    Key=cost-center,Value=CC-BE-001 \
    Key=backup,Value=true \
    Key=owner,Value=backend-team@company.com

# Gắn tags vào EBS snapshot
aws ec2 create-tags \
  --resources snap-xxxxxxxxxxxxxxxxx \
  --tags \
    Key=env,Value=prod \
    Key=source-volume,Value=vol-xxxxxxxxxxxxxxxxx \
    Key=backup-date,Value=2026-05-16
```

### Tag Cho EFS File Systems

```bash
# Gắn tags vào EFS filesystem
aws efs tag-resource \
  --resource-id fs-xxxxxxxx \
  --tags \
    Key=env,Value=prod \
    Key=team,Value=backend \
    Key=project,Value=shared-content \
    Key=cost-center,Value=CC-BE-001
```

### Tag Kế Thừa — Tag Inheritance

```
Vấn đề: S3 object KHÔNG kế thừa tag từ bucket
  Bucket có tag: env=prod
  Object trong bucket: KHÔNG có tag env=prod

Giải pháp cho object tagging:
1. Tag object khi upload
   aws s3 cp file.zip s3://bucket/ --tagging "env=prod&team=backend"

2. S3 Batch Operations để tag hàng loạt object
   (Dùng khi cần retroactively tag — gắn tag ngược cho object cũ)

3. Lifecycle rule + Lambda để tag object khi tạo
   (EventBridge S3 event → Lambda → Tag object)

Note: Chi phí S3 object không thể phân loại theo tag — 
cost allocation tag cho S3 hoạt động ở bucket level, không phải object level.
```

---

## 4. Kích Hoạt Cost Allocation Tags

### Bước 1: Activate Tags Trong Billing Console

```
1. Mở AWS Management Console
2. Vào Billing → Cost Allocation Tags
3. Chọn tab "User-Defined Cost Allocation Tags"
4. Tìm tag key muốn activate
5. Click "Activate"

Lưu ý quan trọng:
  - Phải là root account hoặc IAM user có quyền Billing
  - Chỉ tags đã được activate mới xuất hiện trong Cost Explorer
  - Có thể mất 24 giờ để cost data được cập nhật sau khi activate
  - Tags tạo TRƯỚC khi activate sẽ không có historical data (dữ liệu lịch sử)
```

### Bước 2: Verify Tags Đã Active

```bash
# Kiểm tra tags đã được activate (cần Billing API access)
aws ce list-cost-allocation-tags \
  --status ACTIVE \
  --query 'CostAllocationTags[*].{Key:TagKey,Type:Type,Status:Status}' \
  --output table
```

---

## 5. Phân Tích Chi Phí Với Cost Explorer

### Xem Chi Phí Theo Tag

```bash
# Chi phí theo team (tag key: "team")
aws ce get-cost-and-usage \
  --time-period Start=2026-04-01,End=2026-05-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by '[
    {"Type": "TAG", "Key": "team"},
    {"Type": "DIMENSION", "Key": "SERVICE"}
  ]'

# Kết quả ví dụ:
# Team=backend, SERVICE=Amazon S3 → $1,234
# Team=backend, SERVICE=Amazon EC2 EBS → $567
# Team=data, SERVICE=Amazon S3 → $8,901
```

### Budget Alerts Theo Tag — Cảnh Báo Ngân Sách

```bash
# Tạo budget cảnh báo khi team backend vượt $2,000/tháng
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "BackendTeam-Monthly",
    "BudgetLimit": {
      "Amount": "2000",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["team$backend"]
    }
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{
      "SubscriptionType": "EMAIL",
      "Address": "backend-lead@company.com"
    }]
  }]'
```

### Cost Explorer Dashboard — Bảng Điều Khiển Chi Phí

```
Cách dùng Cost Explorer UI để phân tích:

1. Mở Cost Explorer trong Billing Console
2. Group by: Tag → chọn "team"
3. Filter: Service → "Amazon S3" (để xem chỉ S3 cost)
4. Date range: Last 3 months (3 tháng gần nhất)
5. Granularity: Monthly

→ Biểu đồ hiện chi phí S3 từng team theo tháng
→ Dễ thấy team nào tăng chi phí đột biến
```

---

## 6. Tag Enforcement

### Dùng AWS Config Để Enforce Tags — Kiểm Tra Bắt Buộc Tag

```bash
# Config rule: Phát hiện S3 bucket không có tag "env"
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-must-have-env-tag",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    },
    "InputParameters": "{\"tag1Key\": \"env\", \"tag2Key\": \"team\", \"tag3Key\": \"project\"}"
  }'
```

### Dùng SCP — Service Control Policy — Chính Sách Kiểm Soát Dịch Vụ

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCreateBucketWithoutRequiredTags",
      "Effect": "Deny",
      "Action": "s3:CreateBucket",
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/env": "true",
          "aws:RequestTag/team": "true"
        }
      }
    }
  ]
}
```

```
SCP áp dụng ở Organization level — từ chối tạo S3 bucket
nếu không kèm tag "env" và "team" trong request.

Kết quả: Không thể tạo bucket không có tag → Enforce tagging từ đầu.
```

### Tag Policy Trong AWS Organizations

```json
{
  "tags": {
    "env": {
      "tag_key": {"@@assign": "env"},
      "tag_value": {
        "@@assign": ["prod", "staging", "dev", "test"]
      },
      "enforced_for": {
        "@@assign": [
          "s3:bucket",
          "ec2:volume",
          "elasticfilesystem:file-system"
        ]
      }
    },
    "team": {
      "tag_key": {"@@assign": "team"},
      "tag_value": {
        "@@assign": ["backend", "frontend", "data", "devops", "security"]
      },
      "enforced_for": {
        "@@assign": ["s3:bucket", "ec2:volume"]
      }
    }
  }
}
```

---

## 7. Thực Hành: Tag Policy Cho Tổ Chức

### Template Tag Policy Đề Xuất

```
MANDATORY TAGS (Bắt Buộc) — Tất cả storage resources:
  env          → prod | staging | dev | test
  team         → tên team (backend, frontend, data, devops, security)
  project      → tên project (lowercase, dấu gạch ngang)
  cost-center  → mã trung tâm chi phí (theo chuẩn tài chính công ty)

RECOMMENDED TAGS (Khuyến Nghị) — Nếu có thể:
  owner        → email người phụ trách
  managed-by   → terraform | cloudformation | cdk | manual
  backup       → true | false (có cần backup không)
  data-class   → public | internal | confidential | restricted

STORAGE-SPECIFIC TAGS (Đặc Thù Lưu Trữ):
  S3 bucket:
    purpose    → assets | logs | backup | data-lake | terraform-state
    retention  → 30d | 90d | 1y | 7y | permanent
  
  EBS volume:
    attached-to  → instance-id hoặc hostname
    disk-type    → os | data | swap
  
  EFS:
    access-type  → shared | restricted
    mount-point  → /mnt/shared-content (đường dẫn mount)
```

### Script Kiểm Tra Tag Compliance

```bash
#!/bin/bash
# Check S3 buckets thiếu required tags

REQUIRED_TAGS=("env" "team" "project" "cost-center")

echo "=== S3 Bucket Tag Compliance Report ==="
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
while read bucket; do
  # Lấy tags của bucket
  tags=$(aws s3api get-bucket-tagging --bucket "$bucket" 2>/dev/null \
    --query 'TagSet[*].Key' --output text)
  
  missing_tags=()
  for required in "${REQUIRED_TAGS[@]}"; do
    if ! echo "$tags" | grep -q "$required"; then
      missing_tags+=("$required")
    fi
  done
  
  if [ ${#missing_tags[@]} -gt 0 ]; then
    echo "❌ $bucket — THIẾU: ${missing_tags[*]}"
  else
    echo "✅ $bucket — OK"
  fi
done
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Cost Allocation Tags hoạt động như thế nào trong AWS?**

> Cost Allocation Tags là key-value pairs gắn vào AWS resources cho phép phân loại chi phí. Khi được activate trong Billing Console, chúng xuất hiện như các cột trong cost reports và Cost Explorer, cho phép phân tích chi phí theo team, project hay environment. Tag không tự giảm chi phí mà cung cấp visibility để tìm ra cơ hội tối ưu và implement chargeback nội bộ.

**Q: Tại sao cần activate Cost Allocation Tags trong Billing Console, chỉ tạo tag trên resource chưa đủ?**

> Chỉ tạo tag trên resource không đủ vì AWS không tự động đưa tất cả tags vào billing reports — điều đó sẽ làm báo cáo chi phí quá lớn. Phải activate riêng từng tag key trong Billing Console > Cost Allocation Tags, sau đó mới xuất hiện trong Cost Explorer và billing exports. Mất 24 giờ để dữ liệu được cập nhật và historical data trước khi activate không có sẵn.

**Q: Làm thế nào enforce tagging để team không bỏ sót tag?**

> Ba cách thường dùng kết hợp: (1) SCP — Service Control Policy từ chối tạo resource nếu không có required tags trong request; (2) AWS Config rules tự động phát hiện resource thiếu tag và đánh dấu non-compliant; (3) Tag Policies trong AWS Organizations định nghĩa allowed values cho từng tag key. Ngoài ra, IaC — Infrastructure as Code — Cơ Sở Hạ Tầng Dưới Dạng Code (Terraform, CDK) nên có tagging module bắt buộc để mọi resource đều có đủ tags khi tạo.

**Q: S3 object có thể phân loại chi phí theo tag không?**

> Cost allocation với tags hoạt động ở bucket level cho S3, không phải object level. Không thể phân tích chi phí "object này của team backend, object kia của team frontend" trong cùng một bucket. Giải pháp là mỗi team/project có bucket riêng với tags ở bucket level, hoặc dùng S3 Storage Lens để có analytics chi tiết hơn theo prefix.

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
