# Macie — Phát Hiện và Phân Loại Dữ Liệu Nhạy Cảm Trong S3

> AWS Macie là dịch vụ bảo mật dữ liệu dùng Machine Learning (Học Máy) để tự động phát hiện, phân loại và bảo vệ dữ liệu nhạy cảm được lưu trong Amazon S3. Đặc biệt hiệu quả cho việc phát hiện PII (Personally Identifiable Information — Thông Tin Cá Nhân Có Thể Nhận Dạng) và PHI (Protected Health Information — Thông Tin Sức Khỏe Được Bảo Vệ).

## 📚 Mục Lục

1. [Cách Hoạt Động](#cách-hoạt-động)
2. [Data Identifiers — Bộ Nhận Dạng Dữ Liệu](#data-identifiers)
3. [Sensitive Data Findings — Phát Hiện Dữ Liệu Nhạy Cảm](#sensitive-data-findings)
4. [Policy Findings — Phát Hiện Vi Phạm Chính Sách](#policy-findings)
5. [S3 Bucket Inventory & Posture](#s3-bucket-inventory--posture)
6. [Classification Jobs — Công Việc Phân Loại](#classification-jobs)
7. [Multi-Account Setup](#multi-account-setup)
8. [Tích Hợp Với GDPR & Compliance](#tích-hợp-với-gdpr--compliance)
9. [Chi Phí & Tối Ưu](#chi-phí--tối-ưu)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Cách Hoạt Động

```
S3 Buckets
    │
    ├── Metadata analysis (liên tục, tự động)
    │       ├── Bucket policies (có public access không?)
    │       ├── ACLs (có public ACL không?)
    │       ├── Encryption status (có SSE không?)
    │       └── Replication settings
    │
    └── Content analysis (qua Classification Jobs)
            ├── Macie samples S3 objects (lấy mẫu)
            ├── Phân tích nội dung với ML + pattern matching
            └── So khớp với Managed & Custom Data Identifiers
                    │
                    ▼
            Findings → Security Hub → EventBridge
```

### Phạm Vi Hoạt Động

- **Chỉ S3** — Macie không scan EBS volumes, RDS, DynamoDB, hay Redshift
- Phân tích metadata **liên tục** (không cần cấu hình)
- Phân tích **nội dung** (content) chỉ khi chạy Classification Job

---

## Data Identifiers — Bộ Nhận Dạng Dữ Liệu

### Managed Data Identifiers (Bộ Nhận Dạng Quản Lý Sẵn)

AWS cung cấp 100+ managed data identifiers để phát hiện các loại dữ liệu nhạy cảm phổ biến:

#### Thông Tin Cá Nhân (PII)

| Identifier | Mô Tả | Ví Dụ |
|---|---|---|
| `NAME` | Tên người | John Smith |
| `EMAIL_ADDRESS` | Địa chỉ email | user@example.com |
| `PHONE_NUMBER` | Số điện thoại | +84 912 345 678 |
| `ADDRESS` | Địa chỉ thực | 123 Main St, New York |
| `DATE_OF_BIRTH` | Ngày sinh | 1990-01-15 |
| `PASSPORT_NUMBER` | Số hộ chiếu | US passport format |
| `DRIVER_LICENSE_ID` | Số giấy phép lái xe | Theo format từng tiểu bang |

#### Thông Tin Tài Chính

| Identifier | Mô Tả |
|---|---|
| `CREDIT_CARD_NUMBER` | Số thẻ tín dụng (Visa, MC, Amex, Discover) |
| `CREDIT_CARD_EXPIRY` | Ngày hết hạn thẻ |
| `BANK_ACCOUNT_NUMBER` | Số tài khoản ngân hàng |
| `SWIFT_CODE` | Mã SWIFT/BIC ngân hàng |

#### Thông Tin Y Tế (PHI — Protected Health Information)

| Identifier | Mô Tả |
|---|---|
| `HEALTH_INFORMATION` | Thông tin sức khỏe chung |
| `NATIONAL_IDENTIFICATION_NUMBER` | SSN (Mỹ), NIN (UK)... |

#### Thông Tin Kỹ Thuật

| Identifier | Mô Tả |
|---|---|
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_ACCESS_KEY_ID` | AWS access key ID |
| `OPENSSL_PRIVATE_KEY` | Private key PEM format |
| `HTTP_BASIC_AUTH_HEADER` | Authorization header dạng Basic |
| `USERNAME_AND_PASSWORD` | Credentials dạng text |

### Custom Data Identifiers (Bộ Nhận Dạng Tùy Chỉnh)

Cho phép định nghĩa pattern tùy chỉnh bằng regex để phát hiện dữ liệu đặc thù của tổ chức:

```bash
# Tạo custom identifier để phát hiện mã nhân viên nội bộ
aws macie2 create-custom-data-identifier \
  --name "EmployeeID" \
  --description "Company internal employee ID format EMP-XXXXXX" \
  --regex "EMP-[0-9]{6}" \
  --keywords '["employee", "emp_id", "staff_id"]' \
  --ignore-words '["EXAMPLE-123456", "TEST-000000"]' \
  --maximum-match-distance 50

# Tạo custom identifier để phát hiện số hợp đồng nội bộ
aws macie2 create-custom-data-identifier \
  --name "ContractNumber" \
  --regex "CTR-[0-9]{4}-[A-Z]{2}-[0-9]{4}" \
  --keywords '["contract", "agreement"]'
```

### Allow Lists (Danh Sách Cho Phép)

Định nghĩa patterns hoặc text cụ thể không nên trigger findings:

```bash
# Tạo allow list để bỏ qua test data
aws macie2 create-allow-list \
  --name "TestDataPatterns" \
  --criteria '{
    "regex": "4111-1111-1111-1111|4242-4242-4242-4242"
  }' \
  --description "Known test credit card numbers used in development"
```

---

## Sensitive Data Findings — Phát Hiện Dữ Liệu Nhạy Cảm

### Finding Structure

```json
{
  "type": "SensitiveData:S3Object/Personal",
  "severity": "HIGH",
  "title": "S3 object contains PII",
  "description": "The S3 object contains personal information",
  "resourcesAffected": {
    "s3Bucket": {
      "name": "customer-data-prod",
      "arn": "arn:aws:s3:::customer-data-prod",
      "publicAccess": {
        "effectivePermission": "NOT_PUBLIC"
      },
      "defaultServerSideEncryption": {
        "encryptionType": "AES256"
      }
    },
    "s3Object": {
      "key": "exports/2026-05-16-customer-dump.csv",
      "size": 52428800,
      "serverSideEncryption": {
        "encryptionType": "NONE"
      }
    }
  },
  "classificationDetails": {
    "jobId": "abc123",
    "sensitiveData": [
      {
        "category": "PERSONAL_INFORMATION",
        "detections": [
          {
            "type": "EMAIL_ADDRESS",
            "count": 15420,
            "occurrences": {"lineRanges": [{"start": 2, "end": 15422}]}
          },
          {
            "type": "PHONE_NUMBER",
            "count": 8730
          }
        ],
        "totalCount": 24150
      }
    ]
  }
}
```

### Finding Categories

| Category | Ý Nghĩa |
|---|---|
| `SensitiveData:S3Object/Credentials` | Credentials, API keys, passwords |
| `SensitiveData:S3Object/CustomIdentifier` | Khớp với custom data identifier |
| `SensitiveData:S3Object/Financial` | Thông tin tài chính, thẻ tín dụng |
| `SensitiveData:S3Object/Multiple` | Nhiều loại dữ liệu nhạy cảm |
| `SensitiveData:S3Object/Personal` | PII/PHI |

---

## Policy Findings — Phát Hiện Vi Phạm Chính Sách

Policy findings được tạo **tự động** (không cần Classification Job) từ phân tích metadata S3:

| Finding Type | Mô Tả |
|---|---|
| `Policy:IAMUser/S3BlockPublicAccessDisabled` | IAM user tắt Block Public Access |
| `Policy:IAMUser/S3BucketEncryptionDisabled` | Tắt encryption mặc định của bucket |
| `Policy:IAMUser/S3BucketPublic` | Bucket trở thành public |
| `Policy:IAMUser/S3BucketReplicatedExternally` | Replication đến account ngoài |
| `Policy:IAMUser/S3BucketSharedExternally` | Bucket được chia sẻ ra ngoài org |
| `Policy:IAMUser/S3BucketSharedWithCloudFront` | Bucket chia sẻ với CloudFront ngoài org |

---

## S3 Bucket Inventory & Posture

Macie tự động tạo inventory của tất cả S3 buckets và đánh giá:

```
S3 Bucket Security Posture Dashboard
    ├── Public Access Status
    │       ├── Publicly accessible: X buckets
    │       ├── Not publicly accessible: Y buckets
    │       └── Unknown: Z buckets
    │
    ├── Encryption Status
    │       ├── Encrypted (SSE): A buckets
    │       ├── Not encrypted: B buckets
    │       └── Unknown: C buckets
    │
    ├── Shared Access
    │       ├── Shared externally: D buckets
    │       └── Not shared: E buckets
    │
    └── Sensitive Data Status
            ├── Contains sensitive data: F buckets
            ├── No sensitive data found: G buckets
            └── Not classified: H buckets
```

```bash
# Xem bucket inventory
aws macie2 describe-buckets \
  --criteria '{
    "publicAccess.effectivePermission": {
      "eq": ["PUBLIC"]
    }
  }'

# Xem statistics tổng quan
aws macie2 get-bucket-statistics
```

---

## Classification Jobs — Công Việc Phân Loại

Classification Jobs (Công Việc Phân Loại) là tác vụ phân tích **nội dung** thực sự của S3 objects.

### Loại Jobs

| Loại | Mô Tả | Use Case |
|---|---|---|
| **One-time job** | Chạy một lần duy nhất | Initial scan của bucket mới hoặc historical data |
| **Scheduled job** | Chạy theo lịch (hàng ngày/tuần/tháng) | Ongoing monitoring cho buckets quan trọng |

### Tạo Classification Job

```bash
# Tạo one-time classification job
aws macie2 create-classification-job \
  --job-type ONE_TIME \
  --name "ScanCustomerDataBucket" \
  --s3-job-definition '{
    "bucketDefinitions": [
      {
        "accountId": "123456789012",
        "buckets": ["customer-data-prod", "backup-exports"]
      }
    ],
    "scoping": {
      "includes": {
        "and": [
          {
            "simpleScopeTerm": {
              "comparator": "STARTS_WITH",
              "key": "OBJECT_KEY",
              "values": ["exports/", "reports/"]
            }
          }
        ]
      }
    }
  }' \
  --managed-data-identifier-selector ALL \
  --custom-data-identifier-ids abc123def456

# Tạo scheduled job — chạy hàng tuần
aws macie2 create-classification-job \
  --job-type SCHEDULED \
  --name "WeeklyComplianceScan" \
  --schedule-frequency '{"weeklySchedule": {"dayOfWeek": "MONDAY"}}' \
  --s3-job-definition '{
    "bucketDefinitions": [
      {"accountId": "123456789012", "buckets": ["pci-data-store"]}
    ]
  }' \
  --managed-data-identifier-selector ALL
```

### Sampling Strategy (Chiến Lược Lấy Mẫu)

Macie không scan toàn bộ nội dung mọi file để tiết kiệm chi phí:
- **Mặc định:** scan tối đa 20 MB đầu tiên của mỗi object
- **File archives** (ZIP, GZIP): giải nén và scan từng file bên trong
- **Database exports** (CSV, TSV, ORC, Parquet): scan toàn bộ
- **File types không hỗ trợ:** bỏ qua (binary, compiled code...)

---

## Multi-Account Setup — Thiết Lập Đa Tài Khoản

```bash
# Bước 1: Từ Management Account — ủy quyền Security Account
aws macie2 enable-organization-admin-account \
  --admin-account-id 111122223333

# Bước 2: Từ Security Account — bật Macie cho tất cả members
aws macie2 update-organization-configuration \
  --auto-enable

# Bước 3: Xem findings từ tất cả member accounts
aws macie2 list-findings \
  --finding-criteria '{
    "criterion": {
      "accountId": {
        "neq": []
      }
    }
  }'
```

---

## Tích Hợp Với GDPR & Compliance

### GDPR (General Data Protection Regulation — Quy Định Bảo Vệ Dữ Liệu Chung EU)

Macie giúp đáp ứng GDPR bằng cách:

```
GDPR Article 25 — Privacy by Design
    → Macie phát hiện PII được lưu không có encryption
    → Cảnh báo khi PII ở trong bucket public

GDPR Article 32 — Security of Processing
    → Policy findings cảnh báo khi bucket security bị yếu
    → Audit trail đầy đủ qua CloudTrail

GDPR Article 33 — Breach Notification (72 giờ)
    → Macie + EventBridge + SNS → cảnh báo ngay khi PII bị expose
    → Giảm thời gian phát hiện breach từ ngày → phút
```

### EventBridge Integration Cho Compliance Automation

```python
# Lambda xử lý Macie findings — tự động quarantine bucket
def lambda_handler(event, context):
    findings = event['detail']['findings']
    
    for finding in findings:
        finding_type = finding['type']
        bucket_name = finding['resourcesAffected']['s3Bucket']['name']
        
        # Nếu bucket chứa PII và không có encryption
        if 'SensitiveData' in finding_type:
            encryption = finding['resourcesAffected']['s3Bucket'] \
                .get('defaultServerSideEncryption', {}) \
                .get('encryptionType', 'NONE')
            
            if encryption == 'NONE':
                # Bật SSE-S3 ngay lập tức
                enable_bucket_encryption(bucket_name)
                notify_data_team(bucket_name, finding)
        
        # Nếu bucket PII bị public
        if 'Policy:IAMUser/S3BucketPublic' in finding_type:
            enable_block_public_access(bucket_name)
            create_incident_ticket(finding)

def enable_bucket_encryption(bucket_name):
    s3 = boto3.client('s3')
    s3.put_bucket_encryption(
        Bucket=bucket_name,
        ServerSideEncryptionConfiguration={
            'Rules': [{
                'ApplyServerSideEncryptionByDefault': {
                    'SSEAlgorithm': 'AES256'
                },
                'BucketKeyEnabled': True
            }]
        }
    )
```

---

## Chi Phí & Tối Ưu

| Thành Phần | Giá |
|---|---|
| S3 Bucket Evaluation (liên tục) | Miễn phí |
| Policy Findings | Miễn phí |
| Sensitive data discovery — 30 ngày đầu | Miễn phí (trial) |
| Sensitive data discovery — sau trial | $1.00/GB scanned |
| Automated sensitive data discovery | $0.10/GB processed |

### Giảm Chi Phí Classification Jobs

```bash
# Dùng scoping để chỉ scan file types liên quan
--s3-job-definition '{
  "scoping": {
    "includes": {
      "and": [
        {
          "simpleScopeTerm": {
            "comparator": "STARTS_WITH",
            "key": "OBJECT_EXTENSION",
            "values": [".csv", ".json", ".xlsx", ".pdf"]
          }
        },
        {
          "simpleScopeTerm": {
            "comparator": "GT",
            "key": "OBJECT_SIZE",
            "values": ["1024"]  # Bỏ qua files < 1KB
          }
        }
      ]
    }
  }
}'

# Dùng sampling — chỉ scan percentage của objects
--s3-job-definition '{
  "bucketCriteria": {
    "includes": {
      "and": [
        {
          "simpleCriterion": {
            "comparator": "EQ",
            "key": "S3_BUCKET_SHARED_WITH_AWS_ACCOUNT",
            "values": ["false"]
          }
        }
      ]
    }
  }
}'
```

---

## Câu Hỏi Phỏng Vấn

**Q: Macie có scan EBS, RDS, hay DynamoDB không?**
A: Không. Macie **chỉ scan Amazon S3**. Để phát hiện dữ liệu nhạy cảm trong RDS, cần dùng AWS Glue DataBrew hoặc Comprehend để tùy chỉnh. Đây là giới hạn quan trọng cần nhớ.

**Q: Macie khác Inspector như thế nào?**
A: Inspector phát hiện **security vulnerabilities** (lỗ hổng phần mềm — CVEs). Macie phát hiện **sensitive data** (dữ liệu nhạy cảm như PII, credentials). Hoàn toàn khác nhau về mục đích: Inspector cho software security, Macie cho data security posture.

**Q: Managed Data Identifiers vs Custom Data Identifiers — khi nào dùng cái nào?**
A: Managed Data Identifiers (100+ loại sẵn có) đủ cho hầu hết trường hợp phổ biến (email, SSN, credit card). Custom Data Identifiers cần khi có dữ liệu đặc thù của tổ chức — ví dụ: mã khách hàng nội bộ, số hợp đồng riêng, hay format ID bảo hiểm đặc biệt.

**Q: Macie phát hiện AWS credentials lộ trong S3 bucket như thế nào?**
A: Macie dùng managed data identifier `AWS_SECRET_ACCESS_KEY` và `AWS_ACCESS_KEY_ID` để phát hiện pattern của AWS credentials trong nội dung file. Nếu developer commit file `.env` chứa `AWS_SECRET_ACCESS_KEY=...` lên S3, Macie sẽ tạo finding `SensitiveData:S3Object/Credentials` và cảnh báo ngay.

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
