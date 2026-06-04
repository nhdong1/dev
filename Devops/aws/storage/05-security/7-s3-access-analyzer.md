# S3 Access Analyzer — Phân Tích Quyền Truy Cập

> S3 Access Analyzer — Bộ Phân Tích Quyền Truy Cập S3: tính năng trong AWS IAM Access Analyzer tự động phân tích bucket policies, ACLs, và Access Point policies để phát hiện và cảnh báo khi S3 resources được chia sẻ với bên ngoài tài khoản AWS hoặc ra công khai.

## 📚 Mục Lục

1. [Tổng Quan S3 Access Analyzer](#1-tổng-quan-s3-access-analyzer)
2. [Cách Hoạt Động](#2-cách-hoạt-động)
3. [Thiết Lập Access Analyzer](#3-thiết-lập-access-analyzer)
4. [Phân Tích Findings — Kết Quả Phân Tích](#4-phân-tích-findings)
5. [Access Analyzer for S3 Preview](#5-access-analyzer-for-s3-preview)
6. [Tích Hợp Với Dịch Vụ Khác](#6-tích-hợp-với-dịch-vụ-khác)
7. [Best Practices](#7-best-practices)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan S3 Access Analyzer

### Vấn Đề Cần Giải Quyết

```
Trong một tổ chức có hàng trăm S3 buckets:
- Làm sao biết bucket nào bị public vô tình?
- Làm sao biết bucket nào đang chia sẻ với account bên ngoài?
- Làm sao detect ngay khi policy thay đổi gây ra lỗ hổng bảo mật?

Trước đây: kiểm tra thủ công từng bucket → không scalable
Giải pháp: S3 Access Analyzer → tự động, liên tục, scalable
```

### Hai Dạng Analyzer

| Loại | Phạm Vi Phát Hiện | Dùng Khi |
|------|-----------------|---------|
| **Account Analyzer** | Bucket chia sẻ ngoài account | Kiểm tra từng account độc lập |
| **Organization Analyzer** | Bucket chia sẻ ngoài Organization | Tổ chức dùng AWS Organizations |

### Những Gì Access Analyzer Phân Tích

```
Access Analyzer kiểm tra:
✅ Bucket policies — chính sách bucket
✅ Bucket ACLs — Access Control Lists
✅ S3 Access Point policies — chính sách access point
✅ S3 Multi-Region Access Point policies

Access Analyzer KHÔNG kiểm tra:
❌ IAM identity policies (đó là IAM Access Analyzer)
❌ Object-level ACLs
❌ Object tagging
❌ Pre-signed URLs
```

---

## 2. Cách Hoạt Động

### Cơ Chế Phân Tích

```
Thay đổi policy xảy ra
        │
        ▼
AWS Event (EventBridge)
        │
        ▼
Access Analyzer nhận event
        │
        ▼
Phân tích policy bằng automated reasoning
(Chứng minh toán học, không phải pattern matching)
        │
        ├── Có external access? ──▶ Tạo Finding mới / Cập nhật existing
        │
        └── Không có external access? ──▶ Đánh dấu Active finding là Resolved
```

### Automated Reasoning — Lập Luận Tự Động

Access Analyzer dùng **Zelkova** — công cụ phân tích toán học (formal verification) để chứng minh chính xác ai có thể truy cập resource trong điều kiện nào. Khác với pattern matching đơn giản, approach này đảm bảo không có false negative — không bỏ sót lỗ hổng thực sự.

### Loại Findings

| Finding Type | Mô Tả | Mức Độ |
|-------------|--------|--------|
| **Public** | Bucket accessible bởi bất kỳ ai | CRITICAL |
| **Cross-account** | Bucket chia sẻ với account AWS khác | WARNING |
| **Cross-organization** | Bucket chia sẻ ngoài Organization | WARNING |
| **Cross-service** | Bucket accessible bởi AWS service khác account | INFO |

---

## 3. Thiết Lập Access Analyzer

### 3.1. Bật Qua Console

```
1. Mở IAM Console → Access Analyzer
2. Nhấn "Create analyzer"
3. Chọn loại:
   - Account (Zone of Trust = tài khoản AWS này)
   - Organization (Zone of Trust = toàn bộ Organization)
4. Đặt tên
5. Nhấn "Create analyzer"
```

### 3.2. Bật Qua AWS CLI

```bash
# Tạo Account-level Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name "account-analyzer" \
  --type ACCOUNT \
  --tags Environment=production,Team=security

# Tạo Organization-level Analyzer (cần chạy từ Management Account)
aws accessanalyzer create-analyzer \
  --analyzer-name "org-analyzer" \
  --type ORGANIZATION \
  --tags Environment=production

# Xem danh sách analyzers
aws accessanalyzer list-analyzers

# Bắt đầu quét ngay lập tức
aws accessanalyzer start-resource_scan \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --resource-arn arn:aws:s3:::my-bucket
```

### 3.3. CloudFormation

```yaml
Resources:
  S3AccessAnalyzer:
    Type: AWS::AccessAnalyzer::Analyzer
    Properties:
      AnalyzerName: production-s3-analyzer
      Type: ACCOUNT
      Tags:
        - Key: Environment
          Value: production
        - Key: ManagedBy
          Value: CloudFormation
```

---

## 4. Phân Tích Findings

### 4.1. Cấu Trúc Một Finding

```json
{
  "id": "finding-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "type": "AWS::S3::Bucket",
  "resourceOwnerAccount": "123456789012",
  "resource": "arn:aws:s3:::my-bucket",
  "status": "ACTIVE",
  "createdAt": "2026-05-16T10:00:00Z",
  "updatedAt": "2026-05-16T10:00:00Z",
  "isPublic": true,
  "principal": {
    "AWS": "*"
  },
  "action": ["s3:GetObject", "s3:ListBucket"],
  "condition": {},
  "sources": [
    {
      "type": "BUCKET_POLICY",
      "detail": {}
    }
  ]
}
```

### 4.2. Truy Vấn Findings Qua CLI

```bash
# Liệt kê tất cả findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer

# Lọc chỉ findings đang ACTIVE
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --filter '{"status": {"eq": ["ACTIVE"]}}'

# Lọc findings public (isPublic = true)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --filter '{"isPublic": {"eq": ["true"]}}'

# Lọc findings cho S3
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --filter '{"resourceType": {"eq": ["AWS::S3::Bucket"]}}'

# Xem chi tiết một finding
aws accessanalyzer get-finding \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --id finding-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 4.3. Trạng Thái Findings

| Trạng Thái | Ý Nghĩa | Hành Động |
|-----------|---------|---------|
| **ACTIVE** | Lỗ hổng đang tồn tại | Cần xử lý ngay |
| **ARCHIVED** | Đã archive thủ công | Đã quyết định chấp nhận rủi ro |
| **RESOLVED** | Lỗ hổng đã được sửa | Không cần làm gì |

### 4.4. Xử Lý Findings

```bash
# Archive finding (đã review, chấp nhận intentional access)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --ids '["finding-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"]' \
  --status ARCHIVED

# Bulk archive nhiều findings (ví dụ: tất cả findings đã được review)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --filter '{"status": {"eq": ["ACTIVE"]}}' \
  --query 'findings[*].id' \
  --output text | tr '\t' '\n' | while read finding_id; do
    aws accessanalyzer update-findings \
      --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
      --ids "[$finding_id]" \
      --status ARCHIVED
  done
```

---

## 5. Access Analyzer for S3 Preview

### Policy Validation — Kiểm Tra Policy Trước Khi Áp Dụng

```bash
# Kiểm tra policy mới trước khi áp dụng vào bucket
aws accessanalyzer validate-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }]
  }' \
  --policy-type RESOURCE_POLICY

# Kết quả trả về warnings:
# {
#   "findings": [{
#     "findingType": "WARNING",
#     "issueCode": "ALLOWS_PUBLIC_READ_ACCESS",
#     "learnMoreLink": "...",
#     "locations": [...]
#   }]
# }
```

### Check Access Preview

```bash
# Tạo preview để kiểm tra ai có quyền gì nếu policy thay đổi
aws accessanalyzer create-access-preview \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer \
  --configurations '{
    "arn:aws:s3:::my-bucket": {
      "s3Bucket": {
        "bucketPolicy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::my-bucket/*\"}]}"
      }
    }
  }'

# Xem kết quả preview
aws accessanalyzer list-access-preview-findings \
  --access-preview-id preview-xxxxxxxxxx \
  --analyzer-arn arn:aws:access-analyzer:ap-southeast-1:123456789012:analyzer/account-analyzer
```

---

## 6. Tích Hợp Với Dịch Vụ Khác

### 6.1. EventBridge — Cảnh Báo Tự Động

```json
// EventBridge rule: khi có finding mới từ Access Analyzer
{
  "source": ["aws.access-analyzer"],
  "detail-type": ["Access Analyzer Finding"],
  "detail": {
    "status": ["ACTIVE"],
    "resourceType": ["AWS::S3::Bucket"],
    "isPublic": [true]
  }
}
```

```bash
# Tạo EventBridge rule cảnh báo S3 bucket public
aws events put-rule \
  --name "S3PublicBucketAlert" \
  --event-pattern '{
    "source": ["aws.access-analyzer"],
    "detail-type": ["Access Analyzer Finding"],
    "detail": {
      "status": ["ACTIVE"],
      "isPublic": [true]
    }
  }' \
  --state ENABLED

# Target: SNS topic để gửi email/Slack
aws events put-targets \
  --rule "S3PublicBucketAlert" \
  --targets '[{
    "Id": "SendToSNS",
    "Arn": "arn:aws:sns:ap-southeast-1:123456789012:SecurityAlerts"
  }]'
```

### 6.2. AWS Security Hub

Access Analyzer tự động gửi findings lên Security Hub nếu tích hợp được bật. Findings xuất hiện trong Security Hub dashboard với severity mapping:

| Finding Type | Security Hub Severity |
|-------------|----------------------|
| Public bucket | CRITICAL |
| Cross-account (unintended) | HIGH |
| Cross-organization | MEDIUM |

### 6.3. AWS Config

```bash
# Config rule kiểm tra S3 bucket có phải public
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
  }'

# Rule này chạy khi bucket policy thay đổi — bổ sung cho Access Analyzer
```

### 6.4. Lambda Tự Động Sửa

```python
import boto3
import json

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """Tự động chặn public access khi Access Analyzer phát hiện."""
    
    detail = event.get('detail', {})
    
    # Chỉ xử lý S3 bucket findings đang public
    if (detail.get('resourceType') != 'AWS::S3::Bucket' or
        not detail.get('isPublic', False)):
        return
    
    # Lấy tên bucket từ resource ARN
    resource_arn = detail.get('resource', '')
    bucket_name = resource_arn.split(':::')[-1]
    
    print(f"Phát hiện bucket public: {bucket_name}. Đang chặn truy cập...")
    
    # Bật Block Public Access
    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )
    
    print(f"Đã chặn public access cho bucket: {bucket_name}")
    
    # Gửi thông báo
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:ap-southeast-1:123456789012:SecurityAlerts',
        Subject=f'AUTO-REMEDIATED: S3 Public Bucket {bucket_name}',
        Message=f'Bucket {bucket_name} đã bị tự động chặn public access.\n'
                f'Finding ID: {detail.get("id")}\n'
                f'Vui lòng review ngay để xác nhận.'
    )
```

---

## 7. Best Practices

### 7.1. Checklist Triển Khai

```
□ Bật Organization Analyzer nếu dùng AWS Organizations
□ Bật EventBridge rule cảnh báo findings ACTIVE
□ Tích hợp với Security Hub
□ Đặt SLA xử lý findings:
  - PUBLIC bucket: xử lý trong 1 giờ
  - CROSS-ACCOUNT: review trong 24 giờ
□ Archive findings sau khi review (đừng để tích tụ)
□ Dùng validate-policy trong CI/CD pipeline
□ Review findings định kỳ hàng tuần
```

### 7.2. Tích Hợp Vào CI/CD

```bash
#!/bin/bash
# validate-bucket-policy.sh — chạy trong CI/CD trước khi deploy

POLICY_FILE=$1
BUCKET_NAME=$2

# Validate policy
RESULT=$(aws accessanalyzer validate-policy \
  --policy-document file://$POLICY_FILE \
  --policy-type RESOURCE_POLICY \
  --query 'findings[?findingType==`SECURITY_WARNING`]' \
  --output text)

if [ -n "$RESULT" ]; then
  echo "❌ SECURITY WARNING trong bucket policy:"
  echo "$RESULT"
  exit 1
fi

echo "✅ Policy validation passed"
```

### 7.3. Phân Biệt Intentional vs Accidental Access

```
Intentional (chủ ý) → Archive finding với note:
  - Static website cần public
  - Partner account được cấp quyền chính thức
  - CloudFront distribution được grant

Accidental (vô tình) → Fix ngay:
  - Bucket bị public do misconfiguration
  - Cross-account access không được approved
  - Policy quá rộng do copy-paste sai
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: S3 Access Analyzer khác gì AWS Trusted Advisor S3 bucket checks?**

A: Trusted Advisor kiểm tra đơn giản dựa trên ACL và bucket policy public settings — chủ yếu là pattern matching. Access Analyzer dùng automated reasoning — Zelkova để chứng minh toán học chính xác ai có quyền gì, kể cả trong các điều kiện phức tạp (VPC, IP restrictions, cross-account). Access Analyzer cũng phát hiện cross-account access, không chỉ public access, và tích hợp với EventBridge để cảnh báo realtime.

---

**Q: Khi nào nên archive một finding thay vì fix nó?**

A: Archive khi access là intentional — có chủ ý và đã được approve: ví dụ static website cần public read, partner integration chính thức với cross-account access, hoặc CloudFront serving content từ bucket. Mỗi archive nên có documentation lý do. Fix khi access là không mong muốn, không được approve, hoặc có thể đạt được mục đích tương tự theo cách an toàn hơn (ví dụ dùng OAC thay vì bucket public).

---

**Q: Làm sao phát hiện nếu ai đó thay đổi bucket policy lúc nửa đêm và gây ra lỗ hổng?**

A: Kết hợp ba lớp: (1) CloudTrail ghi lại PutBucketPolicy API call với timestamp và principal — cho biết ai thay đổi lúc mấy giờ. (2) Access Analyzer tự động detect thay đổi policy và tạo finding trong vài phút. (3) EventBridge rule trigger SNS notification hoặc Lambda remediation ngay khi có finding ACTIVE. Với pipeline này, phát hiện và response thường trong 5-15 phút sau khi policy thay đổi.

---

**Cập Nhật:** 2026-05-16
