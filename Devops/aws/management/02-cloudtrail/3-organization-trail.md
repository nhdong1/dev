# CloudTrail Organization Trail — Ghi Nhật Ký Tập Trung Đa Tài Khoản

> **Organization Trail** (Dấu Vết Tổ Chức) là Trail được tạo trong management account và tự động ghi lại event từ **tất cả member accounts** trong AWS Organization — chuẩn thiết kế cho mọi môi trường enterprise multi-account. Không cần cấu hình từng account riêng lẻ.

---

## 📚 Mục Lục

1. [Tại Sao Cần Organization Trail?](#tại-sao-cần-organization-trail)
2. [Kiến Trúc Organization Trail](#kiến-trúc-organization-trail)
3. [So Sánh: Individual Trail vs Organization Trail](#so-sánh-individual-trail-vs-organization-trail)
4. [Cấu Hình Organization Trail](#cấu-hình-organization-trail)
5. [Cấu Trúc S3 Lưu Trữ Đa Account](#cấu-trúc-s3-lưu-trữ-đa-account)
6. [IAM Permissions Cần Thiết](#iam-permissions-cần-thiết)
7. [Centralized Log Analysis — Phân Tích Log Tập Trung](#centralized-log-analysis--phân-tích-log-tập-trung)
8. [Delegated Administrator cho CloudTrail](#delegated-administrator-cho-cloudtrail)
9. [Tích Hợp Với Security Hub & GuardDuty](#tích-hợp-với-security-hub--guardduty)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Organization Trail?

### Vấn Đề Khi Không Có Organization Trail

Giả sử công ty có 20 AWS accounts (dev, staging, prod, security, billing, mỗi team một account):

```
Không có Organization Trail:
  Account Dev-001:   Phải tạo Trail riêng → S3 bucket riêng → Quản lý riêng
  Account Dev-002:   Phải tạo Trail riêng → S3 bucket riêng → Quản lý riêng
  Account Staging:   Phải tạo Trail riêng → S3 bucket riêng → Quản lý riêng
  Account Prod:      Phải tạo Trail riêng → S3 bucket riêng → Quản lý riêng
  ... (20 accounts × 20 regions = 400 configs riêng lẻ)

Vấn đề:
  ❌ Admin của Account Dev-001 có thể tắt Trail của mình → blind spot
  ❌ Account mới tạo không có Trail cho đến khi cấu hình thủ công
  ❌ Audit log rải rác ở 20 S3 buckets → điều tra sự cố phức tạp
  ❌ Chi phí quản lý cao, dễ sai sót
```

### Giải Pháp: Organization Trail

```
Organization Trail (tạo một lần trong Management Account):
  → Tự động áp dụng cho tất cả 20 accounts hiện tại
  → Tự động áp dụng cho mọi account mới tạo trong tương lai
  → Member accounts KHÔNG THỂ tắt Organization Trail
  → Tất cả logs tập trung vào một S3 bucket
  → Điều tra sự cố: query một nơi duy nhất
```

---

## Kiến Trúc Organization Trail

```
┌──────────────────────────────────────────────────────────────────────┐
│                    AWS ORGANIZATION                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │              MANAGEMENT ACCOUNT (123456789000)                   │ │
│  │                                                                  │ │
│  │   ┌──────────────────────────────────────────────────────────┐  │ │
│  │   │              ORGANIZATION TRAIL                          │  │ │
│  │   │  Name: org-audit-trail                                   │  │ │
│  │   │  Type: Multi-Region                                      │  │ │
│  │   │  IsOrganizationTrail: true                               │  │ │
│  │   └────────────────────────┬─────────────────────────────────┘  │ │
│  │                            │                                     │ │
│  └────────────────────────────┼─────────────────────────────────────┘ │
│                               │                                      │
│          ┌────────────────────┼────────────────────┐                 │
│          │                   │                    │                  │
│          ▼                   ▼                    ▼                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │
│  │  Security     │  │  Production   │  │  Dev Account  │            │
│  │  Account      │  │  Account      │  │  234567890123  │            │
│  │  111122223333 │  │  222233334444 │  │               │            │
│  │               │  │               │  │  Events ghi   │            │
│  │  Events ghi   │  │  Events ghi   │  │  tự động vào  │            │
│  │  tự động vào  │  │  tự động vào  │  │  Org Trail    │            │
│  │  Org Trail    │  │  Org Trail    │  │               │            │
│  └───────────────┘  └───────────────┘  └───────────────┘            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    SECURITY/LOG ARCHIVE ACCOUNT                      │
│                                                                      │
│  ┌─────────────────────────────┐   ┌────────────────────────────┐   │
│  │     S3 BUCKET (Central)     │   │   CLOUDWATCH LOGS          │   │
│  │  audit-logs-org-trail       │   │  /cloudtrail/org           │   │
│  │                             │   │                            │   │
│  │  AWSLogs/                   │   │  Metric Filters → Alarms   │   │
│  │    000000000000/ (mgmt)     │   │  Logs Insights queries     │   │
│  │    111122223333/ (security) │   │                            │   │
│  │    222233334444/ (prod)     │   │                            │   │
│  │    234567890123/ (dev)      │   │                            │   │
│  └─────────────────────────────┘   └────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                  AMAZON ATHENA                                   │ │
│  │  Query ALL accounts từ một nơi duy nhất                         │ │
│  │  "Tìm mọi DeleteBucket trong 30 ngày qua — tất cả accounts"     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## So Sánh: Individual Trail vs Organization Trail

| Tiêu Chí                          | Individual Trail (Từng Account)   | Organization Trail               |
| --------------------------------- | --------------------------------- | -------------------------------- |
| **Phạm vi**                       | Một account duy nhất              | Toàn bộ Organization             |
| **Cấu hình**                      | Từng account riêng lẻ             | Một lần ở management account     |
| **Account mới**                   | Phải cấu hình thủ công            | Tự động áp dụng                  |
| **Member account tắt được không?**| ✅ Có thể tắt                     | ❌ Không thể tắt                 |
| **Tập trung log**                 | Nhiều S3 buckets                  | Một S3 bucket trung tâm          |
| **Điều tra cross-account**        | Phải query nhiều nơi              | Query một nơi duy nhất           |
| **Chi phí quản lý**               | Cao (nhiều config)                | Thấp (một config)                |
| **Phù hợp**                       | Account độc lập, nhỏ              | Enterprise, nhiều account         |

---

## Cấu Hình Organization Trail

### Điều Kiện Tiên Quyết

```
1. AWS Organizations đã được bật (All Features enabled, không phải Consolidated Billing only)
2. Người tạo phải có quyền trong management account:
   - cloudtrail:CreateTrail
   - organizations:EnableAWSServiceAccess
   - organizations:ListAccounts
3. S3 bucket phải ở account nơi trail được tạo (hoặc Log Archive account qua cross-account)
4. Trusted access cho CloudTrail trong Organizations phải được bật
```

### Bật Trusted Access

```bash
# Bật CloudTrail integration với Organizations
aws organizations enable-aws-service-access \
  --service-principal cloudtrail.amazonaws.com

# Kiểm tra
aws organizations list-aws-service-access-for-organization
```

### Tạo Organization Trail qua AWS CLI

```bash
# Bước 1: Tạo S3 bucket trong Log Archive account (nếu cross-account)
# Bucket policy phải cho phép tất cả member accounts ghi

# Bước 2: Tạo Organization Trail
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name org-cloudtrail-logs-central \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation \
  --is-organization-trail \
  --kms-key-id arn:aws:kms:us-east-1:123456789000:key/key-id \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789000:log-group:/cloudtrail/org:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789000:role/CloudTrailOrgRole

# Bước 3: Bật Trail
aws cloudtrail start-logging --name org-audit-trail

# Kiểm tra coverage (tất cả accounts có được cover không)
aws cloudtrail get-trail-status --name org-audit-trail
```

### S3 Bucket Policy Cho Organization Trail

Bucket policy phải cho phép tất cả accounts trong Organization ghi:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-central",
      "Condition": {
        "StringEquals": {
          "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:123456789000:trail/org-audit-trail"
        }
      }
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-central/AWSLogs/o-exampleorgid/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceArn": "arn:aws:cloudtrail:us-east-1:123456789000:trail/org-audit-trail"
        }
      }
    }
  ]
}
```

> **Chú ý:** Khi là Organization Trail, đường dẫn S3 chứa organization ID (`o-exampleorgid`) thay vì account ID.

---

## Cấu Trúc S3 Lưu Trữ Đa Account

Khi Organization Trail ghi log, S3 bucket có cấu trúc:

```
s3://org-cloudtrail-logs-central/
└── AWSLogs/
    └── o-exampleorgid/          ← Organization ID
        ├── 123456789000/         ← Management Account
        │   └── CloudTrail/
        │       └── us-east-1/2026/05/17/
        │           └── *.json.gz
        ├── 111122223333/         ← Security Account
        │   └── CloudTrail/
        │       └── us-east-1/2026/05/17/
        │           └── *.json.gz
        ├── 222233334444/         ← Production Account
        │   └── CloudTrail/
        │       └── ...
        └── 234567890123/         ← Dev Account
            └── CloudTrail/
                └── ...
```

---

## IAM Permissions Cần Thiết

### Quyền Để Tạo Organization Trail

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudtrail:CreateTrail",
        "cloudtrail:StartLogging",
        "cloudtrail:GetTrailStatus",
        "cloudtrail:DescribeTrails"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "organizations:EnableAWSServiceAccess",
        "organizations:ListAccounts",
        "organizations:DescribeOrganization"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateServiceLinkedRole"
      ],
      "Resource": "arn:aws:iam::*:role/aws-service-role/cloudtrail.amazonaws.com/*"
    }
  ]
}
```

### Quyền Member Account Đọc Log Của Mình

Member accounts chỉ đọc được log của chính mình (account ID của họ trong path S3):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-central/AWSLogs/o-orgid/222233334444/*"
    }
  ]
}
```

> Security team (trong Security account) có quyền đọc **toàn bộ** bucket để điều tra cross-account.

---

## Centralized Log Analysis — Phân Tích Log Tập Trung

### Athena — Query Tất Cả Accounts Cùng Lúc

Tạo Athena table bao gồm tất cả accounts:

```sql
CREATE EXTERNAL TABLE org_cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        userName: STRING
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIPAddress STRING,
    errorCode STRING,
    errorMessage STRING,
    requestParameters STRING,
    responseElements STRING,
    recipientAccountId STRING
)
PARTITIONED BY (
    account STRING,
    region STRING,
    year STRING,
    month STRING,
    day STRING
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://org-cloudtrail-logs-central/AWSLogs/o-exampleorgid/'
TBLPROPERTIES ('has_encrypted_data'='true');
```

**Query: Tìm tất cả DeleteBucket trong 7 ngày trên mọi account:**

```sql
SELECT
    userIdentity.arn AS actor,
    userIdentity.accountId AS account,
    eventTime,
    awsRegion,
    requestParameters,
    sourceIPAddress
FROM org_cloudtrail_logs
WHERE eventName = 'DeleteBucket'
  AND year = '2026'
  AND month = '05'
  AND day >= '10'
ORDER BY eventTime DESC;
```

**Query: Phát hiện root account usage trong tổ chức:**

```sql
SELECT
    userIdentity.accountId AS account,
    eventTime,
    eventName,
    awsRegion,
    sourceIPAddress
FROM org_cloudtrail_logs
WHERE userIdentity.type = 'Root'
ORDER BY eventTime DESC
LIMIT 100;
```

**Query: Tìm credential bị leak — IP lạ từ nước ngoài:**

```sql
SELECT DISTINCT
    userIdentity.arn,
    userIdentity.accountId,
    sourceIPAddress,
    COUNT(*) AS event_count
FROM org_cloudtrail_logs
WHERE year = '2026' AND month = '05'
  AND userIdentity.type = 'IAMUser'
  AND sourceIPAddress NOT LIKE '10.%'
  AND sourceIPAddress NOT LIKE '172.16.%'
  AND sourceIPAddress NOT LIKE '192.168.%'
GROUP BY userIdentity.arn, userIdentity.accountId, sourceIPAddress
HAVING COUNT(*) > 100
ORDER BY event_count DESC;
```

### CloudWatch Logs Insights — Real-time Monitoring

Query cross-account từ CloudWatch Logs (nếu tất cả accounts gửi vào cùng Log Group):

```
# Tìm mọi failed login trong 1 giờ qua:
fields eventTime, userIdentity.arn, sourceIPAddress, errorCode
| filter eventName = "ConsoleLogin" and responseElements.ConsoleLogin = "Failure"
| stats count(*) as failed_attempts by userIdentity.arn, sourceIPAddress
| sort failed_attempts desc
| limit 20
```

---

## Delegated Administrator cho CloudTrail

**Delegated Administrator** (Quản Trị Viên Được Ủy Quyền) cho phép một member account (thường là Security account) quản lý Organization Trail — không cần dùng Management account cho mọi việc.

### Thiết Lập Delegated Administrator

```bash
# Từ management account — ủy quyền cho Security account
aws cloudtrail register-organization-delegated-admin \
  --member-account-id 111122223333

# Kiểm tra
aws cloudtrail get-organization-delegated-admin

# Security account giờ có thể:
# - Tạo và quản lý Organization Trails
# - Xem trail status của tất cả accounts
# - Không cần quyền tổng thể của Management account
```

### Tại Sao Nên Dùng Delegated Admin?

```
Management Account → Chỉ dùng cho billing và root-level org tasks
Security Account   → Có quyền quản lý audit, không có quyền billing
                    → Principle of least privilege (Quyền Tối Thiểu Cần Thiết)
                    → Separation of duties (Phân Tách Trách Nhiệm)
```

---

## Tích Hợp Với Security Hub & GuardDuty

Organization Trail thường là data source (nguồn dữ liệu) cho các dịch vụ security khác:

```
Organization Trail (CloudTrail logs)
        │
        ├──→ AWS Security Hub
        │      - CloudTrail-based findings
        │      - Cross-account security score
        │
        ├──→ Amazon GuardDuty
        │      - Phân tích CloudTrail để phát hiện threat
        │      - Compromised credentials
        │      - Unusual API calls
        │
        ├──→ Amazon Detective
        │      - Root cause analysis
        │      - Visual investigation graphs
        │
        └──→ Amazon Macie
               - Phân tích S3 data access patterns
               - Phát hiện sensitive data exposure
```

**GuardDuty + CloudTrail Findings phổ biến:**
- `UnauthorizedAccess:IAMUser/TorIPCaller` — IAM user gọi từ Tor network
- `Recon:IAMUser/MaliciousIPCaller` — Reconnaissance từ IP độc hại
- `CredentialAccess:IAMUser/AnomalousBehavior` — Hành vi bất thường của credential
- `Persistence:IAMUser/UserPermissions` — Thay đổi quyền bất thường

---

## Câu Hỏi Phỏng Vấn

### Q1: Organization Trail có tự động áp dụng cho account mới không?

**Có.** Đây là lợi ích chính. Khi account mới được tạo trong Organization (qua Account Factory, AWS Control Tower, hoặc thủ công), Organization Trail tự động bắt đầu ghi log cho account đó — không cần cấu hình thêm. Điều này đảm bảo không có "audit gap" (khoảng trống kiểm toán) giữa thời điểm tạo account và thời điểm cấu hình Trail thủ công.

---

### Q2: Tại sao member account không thể tắt Organization Trail?

Organization Trail được tạo và quản lý bởi management account. Trong member account, Trail này xuất hiện dưới dạng read-only — không có nút Disable/Delete. Ngay cả admin của member account (kể cả root user của member account) cũng không thể tắt hoặc xóa Trail này.

Đây là thiết kế security intentional (bảo mật có chủ đích) — đảm bảo audit trail luôn hoạt động, kể cả trong trường hợp một account bị compromise (xâm phạm).

---

### Q3: Làm thế nào điều tra sự cố cross-account?

**Tình huống:** EC2 trong Production account bị terminate bởi entity nào đó, cần tìm thủ phạm.

**Bước 1 — Query Athena với Organization Trail data:**
```sql
SELECT userIdentity.arn, userIdentity.accountId, eventTime, sourceIPAddress
FROM org_cloudtrail_logs
WHERE eventName = 'TerminateInstances'
  AND requestParameters LIKE '%i-0abc123456%'
```

**Bước 2 — Trace role assume chain:**
Nếu actor là `AssumedRole`, tìm `AssumeRole` event trước đó để xem ai assume role đó:
```sql
SELECT userIdentity.arn, eventTime, requestParameters
FROM org_cloudtrail_logs
WHERE eventName = 'AssumeRole'
  AND requestParameters LIKE '%EC2TerminateRole%'
  AND eventTime > '2026-05-17T03:00:00Z'
```

**Bước 3 — Kiểm tra IP:**
`sourceIPAddress` trong EC2 action là IP của caller — có thể là IP của người dùng hoặc VPC endpoint.

---

### Q4: Sự khác biệt giữa Organization Trail và nhiều Individual Trails mỗi account?

**Organization Trail:**
- 1 config, tự động bao gồm tất cả (kể cả accounts tương lai)
- Member accounts không tắt được
- Log tập trung — query một nơi
- Ít chi phí quản lý hơn

**Individual Trails mỗi account:**
- Phải cấu hình và maintain từng account
- Admin account có thể tắt Trail của mình
- Log rải rác nhiều nơi
- Linh hoạt hơn (mỗi account cấu hình riêng)

Cho môi trường enterprise với nhiều accounts, Organization Trail là lựa chọn duy nhất thực tế.

---

### Q5: Log Archive account là gì và tại sao cần?

**Log Archive account** là một AWS account chuyên dụng chỉ dùng để lưu trữ audit logs (CloudTrail, Config, VPC Flow Logs...). Design principles:

- **Tối thiểu quyền truy cập:** Chỉ CloudTrail service và security team có quyền đọc
- **Không deploy workload:** Giảm attack surface
- **Immutable storage:** S3 Object Lock để không ai xóa được log
- **Separation of concerns:** Người quản lý production không có quyền vào Log Archive

Đây là pattern bắt buộc trong AWS Control Tower landing zone — Log Archive account và Audit account được tạo tự động.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [2-trails-configuration.md](./2-trails-configuration.md) | [4-cloudtrail-insights.md](./4-cloudtrail-insights.md)
