# 2 — IAM Authentication — Xác Thực Danh Tính Cho Database

> IAM (Identity and Access Management — Quản Lý Danh Tính và Quyền Truy Cập) là dịch vụ kiểm soát ai được làm gì với tài nguyên AWS. Áp dụng đúng IAM cho database giúp loại bỏ hardcoded passwords, tạo audit trail (vết kiểm toán) rõ ràng, và giảm thiểu blast radius (phạm vi ảnh hưởng) khi có sự cố.

---

## 📚 Mục Lục

1. [IAM Fundamentals Cho Database](#1-iam-fundamentals-cho-database)
2. [IAM Roles vs IAM Users](#2-iam-roles-vs-iam-users)
3. [IAM Authentication Cho RDS](#3-iam-authentication-cho-rds)
4. [IAM Policy Cho DynamoDB](#4-iam-policy-cho-dynamodb)
5. [IAM Policy Cho RDS Resource Management](#5-iam-policy-cho-rds-resource-management)
6. [Principle of Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu](#6-principle-of-least-privilege--nguyên-tắc-đặc-quyền-tối-thiểu)
7. [Service-Linked Roles & Resource Policies](#7-service-linked-roles--resource-policies)
8. [IAM Access Analyzer — Phân Tích Quyền Truy Cập](#8-iam-access-analyzer--phân-tích-quyền-truy-cập)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. IAM Fundamentals Cho Database

### Hai Loại Quyền Truy Cập Database Cần Quản Lý

```
1. AWS Control Plane (Tầng Điều Khiển AWS)
   ─────────────────────────────────────────
   Ai được tạo/xóa/modify RDS instances?
   Ai được xem snapshots?
   Ai được thay đổi security group?
   → Kiểm soát bằng: IAM Policies

2. Database Data Plane (Tầng Dữ Liệu Database)
   ─────────────────────────────────────────────
   Ai được đăng nhập vào MySQL?
   Ai được SELECT/INSERT/UPDATE trên table nào?
   → Kiểm soát bằng: Database user accounts + (optionally) IAM Auth
```

### IAM Principal Types — Các Loại Đối Tượng IAM

```
┌─────────────────────────────────────────────────────────┐
│ IAM Principal (Chủ Thể IAM)                             │
├──────────────────┬──────────────────────────────────────┤
│ IAM User         │ Long-term credentials (access key)   │
│                  │ Dùng cho: humans (console/CLI)        │
│                  │ Tránh dùng cho workloads              │
├──────────────────┼──────────────────────────────────────┤
│ IAM Role         │ Temporary credentials (auto-rotate)  │
│                  │ Dùng cho: EC2, Lambda, ECS, services │
│                  │ Best practice cho mọi workload        │
├──────────────────┼──────────────────────────────────────┤
│ Service Account  │ Federated identity từ IdP (Okta, AD) │
│ (SAML/OIDC)      │ SSO cho developers access AWS        │
└──────────────────┴──────────────────────────────────────┘
```

---

## 2. IAM Roles vs IAM Users

### Tại Sao Không Dùng IAM User Cho Application?

```python
# ❌ Cực kỳ nguy hiểm — credentials hardcoded trong code
import boto3
client = boto3.client(
    'dynamodb',
    aws_access_key_id='AKIAIOSFODNN7EXAMPLE',      # lộ → toàn bộ AWS account bị compromise
    aws_secret_access_key='wJalrXUtnFEMI/K7MDENG' # thường bị commit lên Git
)

# ✅ Đúng cách — dùng IAM Role (credentials tự động từ EC2 Instance Metadata)
import boto3
client = boto3.client('dynamodb')  # tự động lấy credentials từ IAM Role của EC2
```

### IAM Role Hoạt Động Như Thế Nào

```
1. EC2 instance được gán IAM Role khi launch
2. EC2 Metadata Service (169.254.169.254) cung cấp temporary credentials
3. Credentials tự động rotate mỗi ~6 giờ
4. SDK/CLI tự động dùng credentials này (không cần config thêm)

EC2 Instance (IAM Role: app-server-role)
    │
    ├── Gọi AWS STS (Security Token Service — Dịch Vụ Token Bảo Mật)
    │   → Nhận: AccessKeyId, SecretAccessKey, SessionToken (hết hạn 6h)
    │
    ├── Dùng credentials gọi: dynamodb:GetItem, dynamodb:PutItem
    └── Dùng credentials gọi: secretsmanager:GetSecretValue
```

### Tạo IAM Role Cho EC2 Application Server

```json
// Trust Policy — ai được assume role này?
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ec2.amazonaws.com"    // EC2 được phép assume role này
    },
    "Action": "sts:AssumeRole"
  }]
}

// Permission Policy — role này được làm gì?
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBAppAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": [
        "arn:aws:dynamodb:ap-southeast-1:123456789012:table/orders",
        "arn:aws:dynamodb:ap-southeast-1:123456789012:table/orders/index/*"
      ]
    },
    {
      "Sid": "SecretsManagerReadDB",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/db/mysql-*"
    }
  ]
}
```

---

## 3. IAM Authentication Cho RDS

### IAM Database Authentication Là Gì?

Thay vì dùng username/password truyền thống để đăng nhập RDS, IAM Database Authentication cho phép dùng **IAM token tạm thời** (15 phút hết hạn) để xác thực. Password không bao giờ được lưu trong database.

### Hỗ Trợ và Điều Kiện

| Engine | Hỗ Trợ IAM Auth | Yêu Cầu |
|--------|----------------|---------|
| MySQL | ✅ | Enable `enable_iam_database_authentication` |
| PostgreSQL | ✅ | Enable `enable_iam_database_authentication` |
| Aurora MySQL | ✅ | Giống MySQL |
| Aurora PostgreSQL | ✅ | Giống PostgreSQL |
| MariaDB | ❌ | Không hỗ trợ |
| Oracle | ❌ | Không hỗ trợ |
| SQL Server | ❌ | Không hỗ trợ |

### Cách Hoạt Động IAM Auth Cho RDS

```
Bước 1: Tạo database user đặc biệt
        CREATE USER 'app_user'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';

Bước 2: IAM Policy cho phép rds-db:connect
        {
          "Effect": "Allow",
          "Action": "rds-db:connect",
          "Resource": "arn:aws:rds-db:ap-southeast-1:123456789012:dbuser:db-XXXXX/app_user"
        }

Bước 3: Application tạo auth token
        aws rds generate-db-auth-token \
          --hostname mydb.cluster-xxx.ap-southeast-1.rds.amazonaws.com \
          --port 3306 \
          --region ap-southeast-1 \
          --username app_user
        → Trả về token (chuỗi dài ~800 ký tự, hết hạn 15 phút)

Bước 4: Connect bằng token thay vì password
        mysql -h mydb.cluster-xxx... -u app_user --password=<TOKEN> --ssl-mode=VERIFY_CA
```

### Code Python Dùng IAM Auth Cho RDS

```python
import boto3
import pymysql
import os

def get_db_connection():
    region = 'ap-southeast-1'
    host = os.environ['DB_HOST']
    user = os.environ['DB_USER']   # database username (không phải IAM user)
    db   = os.environ['DB_NAME']

    # Tạo IAM auth token (không cần AWS credentials riêng — dùng IAM Role của instance)
    client = boto3.client('rds', region_name=region)
    token = client.generate_db_auth_token(
        DBHostname=host,
        Port=3306,
        DBUsername=user,
        Region=region
    )

    # Connect với token làm password
    conn = pymysql.connect(
        host=host,
        user=user,
        password=token,
        db=db,
        ssl={'ca': '/etc/ssl/certs/rds-ca-2019-root.pem'},  # SSL bắt buộc với IAM auth
        connect_timeout=5
    )
    return conn
```

### Ưu và Nhược Điểm IAM Auth

| Ưu Điểm | Nhược Điểm |
|---------|-----------|
| Không có password → không bị lộ password | Token hết hạn 15 phút → code phải handle refresh |
| Token tự động hết hạn → giảm rủi ro | Cần SSL bắt buộc |
| Audit qua CloudTrail: ai connect, khi nào | Thêm latency để tạo token (~50ms) |
| Một IAM Policy quản lý access thay vì DB users | Giới hạn 200 kết nối IAM auth/giây/RDS instance |
| Dễ revoke: xóa IAM permission → mất access ngay | Không hỗ trợ mọi engine |

### Khi Nào Dùng IAM Auth vs Secrets Manager

```
IAM Auth phù hợp khi:
✅ App dùng SDK (Python/Java/Node...) — dễ tích hợp
✅ Muốn audit rõ ràng từng kết nối
✅ Cần zero-password policy (không lưu password ở đâu cả)
✅ Môi trường có 200 connections/giây trở xuống

Secrets Manager phù hợp khi:
✅ Legacy app dùng JDBC/ODBC connection string cũ — khó thay token logic
✅ Engine không hỗ trợ IAM Auth (MariaDB, Oracle, SQL Server)
✅ Cần centralized secret management cho nhiều loại credentials
✅ Connection pool lớn (IAM auth có giới hạn 200/giây)
```

---

## 4. IAM Policy Cho DynamoDB

### Granular Access — Kiểm Soát Chi Tiết

DynamoDB hỗ trợ IAM policy ở cấp độ rất chi tiết:

```json
// Policy 1: Read-only cho table orders, chỉ attributes không nhạy cảm
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DynamoDBReadOnlyOrders",
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:Query",
      "dynamodb:Scan"
    ],
    "Resource": "arn:aws:dynamodb:ap-southeast-1:123456789012:table/orders",
    "Condition": {
      "ForAllValues:StringEquals": {
        "dynamodb:Attributes": [
          "orderId", "status", "createdAt", "totalAmount"
          // Không include: paymentInfo, customerSSN, v.v.
        ]
      },
      "StringEqualsIfExists": {
        "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
      }
    }
  }]
}
```

```json
// Policy 2: App chỉ được đọc/ghi data của chính user mình (row-level security)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DynamoDBUserDataAccess",
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:UpdateItem",
      "dynamodb:DeleteItem",
      "dynamodb:Query"
    ],
    "Resource": "arn:aws:dynamodb:ap-southeast-1:123456789012:table/user-data",
    "Condition": {
      "ForAllValues:StringEquals": {
        "dynamodb:LeadingKeys": ["${aws:userid}"]
        // ${aws:userid} = IAM principal ID — chỉ access partition key = user ID của mình
      }
    }
  }]
}
```

### DynamoDB IAM Actions Quan Trọng

```
Data plane (thường cấp cho applications):
├── dynamodb:GetItem          — đọc 1 item theo key
├── dynamodb:PutItem          — tạo/overwrite item
├── dynamodb:UpdateItem       — update attributes
├── dynamodb:DeleteItem       — xóa item
├── dynamodb:Query            — query theo partition key
├── dynamodb:Scan             — scan toàn bộ table (cẩn thận!)
├── dynamodb:BatchGetItem     — batch read
├── dynamodb:BatchWriteItem   — batch write
└── dynamodb:TransactWriteItems / TransactGetItems — transactions

Management plane (chỉ cấp cho admin/ops):
├── dynamodb:CreateTable
├── dynamodb:DeleteTable
├── dynamodb:UpdateTable
├── dynamodb:DescribeTable
├── dynamodb:ListTables
└── dynamodb:CreateGlobalTable
```

---

## 5. IAM Policy Cho RDS Resource Management

### Phân Biệt Hai Loại IAM Policies Cho RDS

```
1. IAM Policy cho AWS API (Control Plane)
   → Ai được tạo/modify/delete RDS instances?
   → Ai được xem/restore snapshots?
   → Actions: rds:CreateDBInstance, rds:DeleteDBInstance, rds:DescribeDBInstances

2. IAM Policy cho Database Authentication (Data Plane)
   → Ai được đăng nhập vào RDS qua IAM auth?
   → Action: rds-db:connect
   → Resource: arn:aws:rds-db:region:account:dbuser:db-resource-id/db-username
```

### Policy Cho DB Admin (Quản Trị Viên Database)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSFullManagement",
      "Effect": "Allow",
      "Action": [
        "rds:Describe*",
        "rds:List*",
        "rds:CreateDBSnapshot",
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:ModifyDBInstance",
        "rds:RebootDBInstance"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDeleteProduction",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster"
      ],
      "Resource": "arn:aws:rds:*:*:db:prod-*"
      // Không ai được xóa RDS có tên bắt đầu bằng "prod-"
    }
  ]
}
```

### Policy Cho Application (Chỉ Đọc Metadata RDS)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSDescribeOnly",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSIAMAuth",
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:ap-southeast-1:123456789012:dbuser:cluster-ABCDEFGH/app_user"
    }
  ]
}
```

---

## 6. Principle of Least Privilege — Nguyên Tắc Đặc Quyền Tối Thiểu

### Định Nghĩa

**Principle of Least Privilege** (Nguyên Tắc Đặc Quyền Tối Thiểu): Mỗi entity (user, role, service) chỉ được cấp đúng quyền cần thiết để thực hiện công việc của mình — không hơn, không kém.

### Áp Dụng Thực Tế

```
Scenario: E-commerce app với các components sau

ORDER SERVICE (Dịch Vụ Đơn Hàng):
  IAM Role: order-service-role
  Permissions:
  ✅ dynamodb:GetItem, PutItem, UpdateItem ON orders-table
  ✅ dynamodb:Query ON orders-table/OrdersByUser-index
  ❌ KHÔNG có: dynamodb:Scan (không cần toàn table scan)
  ❌ KHÔNG có: dynamodb:DeleteItem (đơn hàng không xóa)
  ❌ KHÔNG có: access vào bất kỳ table nào khác

USER SERVICE (Dịch Vụ Người Dùng):
  IAM Role: user-service-role
  Permissions:
  ✅ dynamodb:GetItem, PutItem, UpdateItem ON users-table
  ✅ secretsmanager:GetSecretValue ON prod/db/user-service-*
  ❌ KHÔNG có: access vào orders-table

ANALYTICS SERVICE (Dịch Vụ Phân Tích):
  IAM Role: analytics-service-role
  Permissions:
  ✅ dynamodb:Scan, Query ON orders-table (read-only)
  ✅ rds-db:connect AS analytics_readonly user
  ❌ KHÔNG có: write permissions
  ❌ KHÔNG có: access vào users-table (GDPR compliance)
```

### Permission Boundary — Ranh Giới Quyền

**Permission Boundary** giới hạn tối đa quyền mà một IAM Role có thể có, kể cả khi được cấp policy rộng hơn:

```json
// Permission Boundary cho tất cả roles trong production
// → Ngay cả admin cũng không thể vượt quá ranh giới này
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:*",
        "dynamodb:*",
        "elasticache:*",
        "secretsmanager:GetSecretValue",
        "kms:Decrypt"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",       // Không được modify IAM
        "organizations:*",
        "account:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### IAM Policy Conditions — Điều Kiện Phổ Biến

```json
// Chỉ cho phép từ VPC cụ thể
"Condition": {
  "StringEquals": {
    "aws:SourceVpc": "vpc-prod123"
  }
}

// Chỉ cho phép trong giờ hành chính (UTC+7 = UTC+0 là 1:00-11:00)
"Condition": {
  "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T01:00:00Z"},
  "DateLessThan":    {"aws:CurrentTime": "2024-12-31T11:00:00Z"}
}

// Bắt buộc MFA để thực hiện thao tác nguy hiểm
"Condition": {
  "Bool": {"aws:MultiFactorAuthPresent": "true"}
}

// Chỉ cho phép từ IP office
"Condition": {
  "IpAddress": {
    "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
  }
}
```

---

## 7. Service-Linked Roles & Resource Policies

### Service-Linked Roles — Vai Trò Liên Kết Dịch Vụ

Một số dịch vụ AWS tự tạo IAM Role riêng để hoạt động:

```
AWSServiceRoleForRDS:
  → AWS RDS dùng role này để:
     - Gọi EC2 API để manage network interfaces
     - Gọi CloudWatch để push metrics
     - Gọi SNS để send event notifications

AWSServiceRoleForElastiCache:
  → ElastiCache dùng để manage VPC resources

→ Bạn không quản lý các role này trực tiếp
→ Chỉ cần biết chúng tồn tại để không xóa nhầm
```

### DynamoDB Resource-Based Policy

DynamoDB hỗ trợ resource-based policy (chính sách dựa trên tài nguyên), cho phép cross-account access:

```json
// Policy attach vào DynamoDB table để cho phép account B read
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "CrossAccountReadAccess",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT-B-ID:role/analytics-role"
    },
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:Query",
      "dynamodb:Scan"
    ],
    "Resource": "arn:aws:dynamodb:ap-southeast-1:ACCOUNT-A-ID:table/shared-data"
  }]
}
```

---

## 8. IAM Access Analyzer — Phân Tích Quyền Truy Cập

### IAM Access Analyzer Là Gì?

Công cụ tự động phân tích IAM policies để tìm:
- Resources exposed bên ngoài tài khoản (external access)
- Unused permissions (quyền không dùng đến)
- Policy violations theo best practices

```bash
# Tạo analyzer cho production account
aws accessanalyzer create-analyzer \
  --analyzer-name prod-access-analyzer \
  --type ACCOUNT  # Kiểm tra cross-account access

# Xem findings (kết quả phân tích)
aws accessanalyzer list-findings --analyzer-name prod-access-analyzer

# Ví dụ finding nguy hiểm:
# DynamoDB table "customer-data" có resource policy
# cho phép arn:aws:iam::*:root (mọi AWS account) read
# → Cần investigate ngay
```

### IAM Policy Validator

```bash
# Kiểm tra policy có lỗi syntax không
aws accessanalyzer validate-policy \
  --policy-document file://my-policy.json \
  --policy-type IDENTITY_POLICY

# Trả về:
# - ERRORS: Policy không valid (sẽ bị reject)
# - SECURITY_WARNINGS: Potential security issues
# - WARNINGS: Best practice violations
# - SUGGESTIONS: Recommendations
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Tại sao không nên dùng IAM User với access key cho application chạy trên EC2?**

> IAM User credentials không hết hạn — nếu bị lộ, attacker có access vô thời hạn. IAM Role cấp temporary credentials (6 giờ), tự động rotate, và không cần lưu credentials ở đâu. Nếu EC2 bị compromise, attacker chỉ có credentials hết hạn trong vài giờ thay vì vĩnh viễn.

**Q: Giải thích IAM Authentication cho RDS — hoạt động thế nào và ưu nhược điểm?**

> IAM Auth cho RDS: thay vì password, application dùng AWS SDK generate auth token (hết hạn 15 phút), token này được dùng làm password khi connect. IAM policy `rds-db:connect` kiểm soát ai được connect. Ưu điểm: không có password tĩnh, audit rõ ràng, revoke dễ. Nhược điểm: chỉ hỗ trợ MySQL/PostgreSQL, giới hạn 200 IAM auth connections/giây/instance.

**Q: Thiết kế IAM policy cho microservices — order-service chỉ được đọc/ghi orders table, không được động vào bất kỳ table nào khác. Làm thế nào?**

> Tạo IAM Role riêng cho order-service với policy chỉ allow `dynamodb:GetItem, PutItem, UpdateItem, Query` trên ARN cụ thể của orders-table (và index của nó). Không dùng wildcard `*` trong Resource. Áp dụng Permission Boundary để đảm bảo ngay cả khi policy bị modify, vẫn không vượt quá giới hạn cho phép.

**Q: Làm sao kiểm tra IAM policies có an toàn không?**

> Dùng IAM Access Analyzer để: (1) tìm resources bị exposed public/cross-account ngoài ý muốn; (2) tìm unused permissions (IAM Access Advisor); (3) validate policy syntax và security warnings. Kết hợp với AWS Config rules để enforce rằng không có resource nào public mà không được approved.

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tiếp → |
|---------|-----------------|--------|
| [1-vpc-security-groups.md](./1-vpc-security-groups.md) | **2-iam-authentication.md** | [3-encryption.md](./3-encryption.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
