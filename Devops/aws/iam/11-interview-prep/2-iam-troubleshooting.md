# IAM Troubleshooting — Debug Access Denied & Policy Conflicts

> Hướng dẫn có hệ thống để debug mọi vấn đề IAM — từ "Access Denied" đơn giản đến xung đột policy phức tạp trong môi trường multi-account.

---

## 📑 Mục Lục

1. [Framework Debug Tổng Quát](#framework-debug-tổng-quát)
2. [Scenario 1: S3 Access Denied](#scenario-1-s3-access-denied)
3. [Scenario 2: Cross-Account Access Thất Bại](#scenario-2-cross-account-access-thất-bại)
4. [Scenario 3: Lambda Không Gọi Được DynamoDB](#scenario-3-lambda-không-gọi-được-dynamodb)
5. [Scenario 4: SCP Chặn Không Mong Muốn](#scenario-4-scp-chặn-không-mong-muốn)
6. [Scenario 5: Permission Boundary Conflict](#scenario-5-permission-boundary-conflict)
7. [Scenario 6: AssumeRole Thất Bại](#scenario-6-assumerole-thất-bại)
8. [Công Cụ Debug IAM](#công-cụ-debug-iam)
9. [Anti-Patterns Phổ Biến](#anti-patterns-phổ-biến)

---

## Framework Debug Tổng Quát

### Quy Trình 5 Bước

```
BƯỚC 1: ĐỌC LỖI CẨN THẬN
└── Xác định: principal, action, resource, và "explicit deny" hay "implicit deny"

BƯỚC 2: XÁC ĐỊNH PRINCIPAL
└── aws sts get-caller-identity
└── Xác định: IAM User, IAM Role, Federated User, hay Service?

BƯỚC 3: TRACE TỪNG LỚP POLICY (theo thứ tự đánh giá)
└── SCP (nếu dùng Organizations)
└── Resource-based Policy (nếu có)
└── Permission Boundary (nếu được set)
└── Identity-based Policy
└── Session Policy (nếu AssumeRole với policy)

BƯỚC 4: KIỂM TRA CONDITIONS
└── MFA required? Time-based? IP-based? Tag-based?

BƯỚC 5: XÁC NHẬN VỚI TOOLS
└── IAM Policy Simulator
└── CloudTrail lookup-events
└── Access Analyzer validate-policy
```

### Phân Biệt Explicit Deny vs Implicit Deny

```bash
# Error có chứa "with an explicit deny" → tìm DENY statement
AccessDenied: ... is not authorized to perform: s3:GetObject 
on resource: arn:aws:s3:::bucket/file 
with an explicit deny

# Error KHÔNG có "explicit deny" → thiếu ALLOW statement (implicit deny)
AccessDenied: ... is not authorized to perform: s3:GetObject 
on resource: arn:aws:s3:::bucket/file
```

---

## Scenario 1: S3 Access Denied

### Tình Huống

IAM user `alice` trong Account A cố gắng đọc object từ S3 bucket trong cùng account. Bucket policy trông có vẻ đúng nhưng vẫn bị denied.

### Bucket Policy (có vẻ đúng)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

### Diagnosis Checklist

```
1. Kiểm tra IAM user policy
   → User có identity policy allow s3:GetObject không?
   → Với S3 same-account: cần có EITHER bucket policy OR user policy (hoặc cả hai)
   → Với cross-account: cần CẢ HAI bucket policy VÀ user policy

2. Kiểm tra S3 Block Public Access (Chặn Truy Cập Công Khai S3)
   aws s3api get-bucket-policy-status --bucket my-bucket
   aws s3api get-public-access-block --bucket my-bucket
   → BlockPublicPolicy: true → từ chối bucket policy có Principal: "*"

3. Kiểm tra bucket ACL (Access Control List — Danh Sách Kiểm Soát Truy Cập)
   aws s3api get-bucket-acl --bucket my-bucket

4. Kiểm tra object ACL
   aws s3api get-object-acl --bucket my-bucket --key secret.txt

5. Kiểm tra KMS key policy (nếu object được mã hóa SSE-KMS)
   → User có kms:Decrypt permission không?
   → Key policy có include user không?

6. Kiểm tra Permission Boundary của user
   aws iam get-user --user-name alice
   → Xem PermissionsBoundary field
```

### Fix Phổ Biến

**Fix 1: Thêm user policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Fix 2: Bucket policy với principal cụ thể thay vì wildcard**
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:user/alice"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Fix 3: Tắt Block Public Access nếu bucket cần public**
```bash
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

### Bài Học

> S3 là dịch vụ đặc biệt: **cùng account** → bucket policy OR IAM policy là đủ; **cross-account** → phải có CẢ HAI.

---

## Scenario 2: Cross-Account Access Thất Bại

### Tình Huống

Application trong Account A (123456789012) cố assume role trong Account B (987654321098) để đọc DynamoDB. Nhận lỗi:

```
An error occurred (AccessDenied) when calling the AssumeRole operation: 
User: arn:aws:iam::123456789012:role/AppRole 
is not authorized to perform: sts:AssumeRole 
on resource: arn:aws:iam::987654321098:role/DynamoDBReadRole
```

### Checklist Debug

```
KIỂM TRA 1: Trust Policy của DynamoDBReadRole trong Account B
→ Principal có include AppRole từ Account A không?
→ Condition có ExternalId requirement không?
→ Principal format đúng không? (AWS: arn:... không phải chỉ account ID)

KIỂM TRA 2: Permission Policy của AppRole trong Account A
→ AppRole có policy allow sts:AssumeRole không?
→ Resource ARN trong policy có đúng không?

KIỂM TRA 3: SCP trong Organizations
→ SCP có deny sts:AssumeRole hoặc sts:* không?
→ SCP có condition nào block cross-account không?

KIỂM TRA 4: Permission Boundary của AppRole
→ Boundary có include sts:AssumeRole không?
```

### Trust Policy Đúng Trong Account B

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:role/AppRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "secret-external-id-xyz"
      }
    }
  }]
}
```

### Permission Policy Cần Có Trong Account A

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::987654321098:role/DynamoDBReadRole"
  }]
}
```

### Lỗi Phổ Biến

```
❌ Principal sai:
"Principal": {"AWS": "987654321098"}  # Thiếu "arn:aws:iam::" prefix

✅ Principal đúng:
"Principal": {"AWS": "arn:aws:iam::123456789012:role/AppRole"}

❌ Quên ExternalId trong code gọi AssumeRole:
aws sts assume-role --role-arn "arn:..." --role-session-name "test"

✅ Đúng khi Trust Policy yêu cầu ExternalId:
aws sts assume-role \
  --role-arn "arn:..." \
  --role-session-name "test" \
  --external-id "secret-external-id-xyz"
```

---

## Scenario 3: Lambda Không Gọi Được DynamoDB

### Tình Huống

Lambda function bị lỗi `AccessDenied` khi gọi `DynamoDB:PutItem`. Execution role đã có `AmazonDynamoDBFullAccess` policy.

### Debug Steps

```bash
# 1. Xem Execution Role của Lambda
aws lambda get-function-configuration \
  --function-name my-function \
  --query 'Role'

# 2. Kiểm tra attached policies
aws iam list-attached-role-policies \
  --role-name MyLambdaRole

# 3. Xem inline policies
aws iam list-role-policies \
  --role-name MyLambdaRole

# 4. Xem Permission Boundary
aws iam get-role --role-name MyLambdaRole \
  --query 'Role.PermissionsBoundary'

# 5. CloudTrail tìm event
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-function \
  --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ)
```

### Nguyên Nhân Phổ Biến

**1. Lambda VPC và DynamoDB không có VPC Endpoint**
```
Lambda trong VPC → không có route ra internet → DynamoDB không accessible
→ Fix: Tạo DynamoDB VPC Gateway Endpoint

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxx \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids rtb-xxxxx
```

**2. Resource-based condition restrict DynamoDB table**
```json
// DynamoDB table có resource policy condition
{
  "Condition": {
    "StringEquals": {
      "aws:SourceVpc": "vpc-production-only"
    }
  }
}
// → Lambda trong dev VPC bị từ chối
```

**3. KMS key policy không include Lambda role**
```
DynamoDB table encrypted với CMK
→ Lambda cần kms:Decrypt permission
→ Key Policy phải include Lambda execution role
```

**4. Permission Boundary của Lambda role**
```json
// Boundary chỉ allow S3, không include DynamoDB
{
  "Effect": "Allow",
  "Action": ["s3:*"],  // ← Thiếu dynamodb:*
  "Resource": "*"
}
```

---

## Scenario 4: SCP Chặn Không Mong Muốn

### Tình Huống

Admin trong child account không thể tạo S3 bucket dù có `AdministratorAccess`. Lỗi:

```
AccessDenied: User: arn:aws:iam::123456789012:user/admin 
is not authorized to perform: s3:CreateBucket
```

### Debug SCPs

```bash
# 1. Liệt kê SCPs áp dụng cho account
aws organizations list-policies-for-target \
  --target-id 123456789012 \
  --filter SERVICE_CONTROL_POLICY

# 2. Xem content của từng SCP
aws organizations describe-policy --policy-id p-xxxxxxxx

# 3. Kiểm tra SCPs ở cấp OU cha (phải làm trong Management Account)
aws organizations list-parents --child-id 123456789012
aws organizations list-policies-for-target \
  --target-id ou-xxxx-xxxxxxxx \
  --filter SERVICE_CONTROL_POLICY
```

### SCP Hay Gây Nhầm Lẫn

```json
// SCP này KHÔNG có allow S3 → deny S3 cho toàn account
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:*",
      "iam:*"
      // Thiếu "s3:*" → s3 bị deny ngầm định
    ],
    "Resource": "*"
  }]
}

// Cần thêm:
{
  "Effect": "Allow",
  "Action": ["s3:*"],
  "Resource": "*"
}
```

### SCP Inheritance (Kế Thừa SCP)

```
Root → SCP-A: Allow ec2:*, s3:*, iam:*
  └── OU-Production → SCP-B: Deny ec2:TerminateInstances
      └── Account → User với AdministratorAccess

User trong Account có quyền:
= (Allow ec2:*, s3:*, iam:*) 
  AND NOT (ec2:TerminateInstances)
= Tất cả EC2 actions TRỪ TerminateInstances, tất cả S3, tất cả IAM
```

> **Quan trọng:** SCP không cấp quyền — chúng chỉ giới hạn quyền tối đa. Identity policy vẫn phải có Allow.

---

## Scenario 5: Permission Boundary Conflict

### Tình Huống

Developer được phép tự tạo IAM Role cho Lambda của họ. Tuy nhiên role tự tạo không hoạt động dù có đúng policy.

### Nguyên Nhân

Developer tạo role mà quên đặt Permission Boundary (theo requirement của org):

```bash
# Admin đã tạo policy cho phép developer tạo role với condition:
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
    }
  }
}
# → Developer PHẢI set boundary khi tạo role
# → Nếu không set → AccessDenied khi tạo role

# Fix: Developer tạo role với boundary
aws iam create-role \
  --role-name MyLambdaRole \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary
```

### Permission Boundary Không Tự Cấp Quyền

```
Developer Boundary cho phép: s3:*, dynamodb:*, logs:*
Lambda Role có policy: s3:*, dynamodb:*, ec2:*

Lambda Role thực sự có quyền:
= (s3:*, dynamodb:*, ec2:*) INTERSECTION (s3:*, dynamodb:*, logs:*)
= s3:*, dynamodb:*
(ec2:* và logs:* bị loại vì không nằm trong intersection)
```

---

## Scenario 6: AssumeRole Thất Bại

### Tình Huống

EC2 instance cố assume một role khác để truy cập tài nguyên trong region khác.

```
Error: Not authorized to perform sts:AssumeRole on arn:aws:iam::123456789012:role/OtherRole
```

### Common Causes & Fixes

**Nguyên nhân 1: Trust Policy không include EC2 instance profile**
```json
// Sai — Trust Policy chỉ allow human users
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/alice"
  }
}

// Đúng — Trust Policy cho EC2
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}

// Đúng — Trust Policy cho EC2 instance profile cụ thể
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/EC2InstanceRole"
  },
  "Action": "sts:AssumeRole"
}
```

**Nguyên nhân 2: Chaining role bị giới hạn (session duration)**
```
EC2 Instance Profile → AssumeRole(RoleA, max 1h) → AssumeRole(RoleB)

Khi chaining: duration của RoleB bị giới hạn bởi duration còn lại của RoleA
Nếu RoleA token sắp hết → RoleB có thể không được cấp đủ thời gian
```

**Nguyên nhân 3: IMDSv2 (Instance Metadata Service v2) bắt buộc nhưng code dùng v1**
```bash
# EC2 instance được enforce IMDSv2
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxxxx \
  --http-tokens required

# Application code dùng IMDSv1 (không có token) → bị chặn
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Fix: Dùng IMDSv2 với token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
```

---

## Công Cụ Debug IAM

### 1. IAM Policy Simulator

```bash
# Simulate policy cho user cụ thể
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names "s3:GetObject" "s3:PutObject" \
  --resource-arns "arn:aws:s3:::my-bucket/*" \
  --context-entries '[
    {"ContextKeyName": "aws:MultiFactorAuthPresent", "ContextKeyValues": ["true"], "ContextKeyType": "boolean"},
    {"ContextKeyName": "aws:SourceIp", "ContextKeyValues": ["10.0.0.1"], "ContextKeyType": "ip"}
  ]'

# Output:
# {
#   "EvaluationResults": [{
#     "EvalActionName": "s3:GetObject",
#     "EvalDecision": "allowed",  # hoặc "explicitDeny" hoặc "implicitDeny"
#     "MatchedStatements": [...]
#   }]
# }
```

### 2. Access Analyzer — Validate Policy

```bash
# Kiểm tra policy có lỗi hay không
aws accessanalyzer validate-policy \
  --policy-document file://my-policy.json \
  --policy-type IDENTITY_POLICY

# Kiểm tra resource policy
aws accessanalyzer validate-policy \
  --policy-document file://bucket-policy.json \
  --policy-type RESOURCE_POLICY

# Output tìm:
# SUGGESTION — Cải thiện
# WARNING — Cảnh báo tiềm ẩn
# ERROR — Lỗi syntax/logic
# SECURITY_WARNING — Cảnh báo bảo mật
```

### 3. CloudTrail — Tìm Event Cụ Thể

```bash
# Tìm AccessDenied events của user
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=alice \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[?contains(CloudTrailEvent, `AccessDenied`)].[EventTime,EventName,CloudTrailEvent]'

# Tìm event theo EventName
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)

# Athena query trong CloudTrail Lake
SELECT
  useridentity.arn,
  eventname,
  errorcode,
  errormessage,
  sourceipaddress,
  eventtime
FROM cloudtrail_logs
WHERE errorcode IN ('AccessDenied', 'UnauthorizedAccess')
  AND eventtime > DATE_ADD('hour', -24, NOW())
ORDER BY eventtime DESC
LIMIT 100;
```

### 4. get-caller-identity — Xác Định Who Am I

```bash
# Khi debug, luôn bắt đầu bằng lệnh này
aws sts get-caller-identity

# Output:
# {
#   "UserId": "AROAXXXXXXXXXXXXXXXXX:my-session",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/MyRole/my-session"
# }
```

### 5. IAM Authorization Details — Snapshot Toàn Bộ IAM

```bash
# Xuất toàn bộ IAM config (users, roles, policies, groups)
aws iam get-account-authorization-details \
  --filter User Role Group LocalManagedPolicy \
  --output json > iam-snapshot.json

# Dùng để phân tích offline hoặc với tools như iamlive, Parliament
```

---

## Anti-Patterns Phổ Biến

### ❌ Anti-Pattern 1: Wildcard Action + Wildcard Resource

```json
// Nguy hiểm — quá rộng
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// Tốt hơn — specific actions và resources
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-specific-bucket/*"
}
```

### ❌ Anti-Pattern 2: Deny với NotAction

```json
// Dễ nhầm — deny mọi thứ TRỪ s3:GetObject (không phải những gì bạn nghĩ)
{
  "Effect": "Deny",
  "NotAction": "s3:GetObject",
  "Resource": "*"
}
// → Deny iam:*, ec2:*, kms:* ... tất cả ngoại trừ s3:GetObject
// → Thường gây "deny quá nhiều" không mong muốn
```

### ❌ Anti-Pattern 3: Allow s3:* Không Có ListBucket

```json
// Khi cần ListObjects, cần 2 permissions riêng biệt:
{
  "Effect": "Allow",
  "Action": "s3:ListBucket",          // list objects trong bucket
  "Resource": "arn:aws:s3:::my-bucket" // → resource là BUCKET (không có /*)
},
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*" // → resource là OBJECTS (có /*)
}
```

### ❌ Anti-Pattern 4: Quên Condition cho Sensitive Actions

```json
// Nguy hiểm — không yêu cầu MFA để delete
{
  "Effect": "Allow",
  "Action": "dynamodb:DeleteTable",
  "Resource": "*"
}

// An toàn hơn — yêu cầu MFA
{
  "Effect": "Allow",
  "Action": "dynamodb:DeleteTable",
  "Resource": "*",
  "Condition": {
    "Bool": {"aws:MultiFactorAuthPresent": "true"},
    "NumericLessThan": {"aws:MultiFactorAuthAge": "300"}  // MFA trong 5 phút gần đây
  }
}
```

### ❌ Anti-Pattern 5: Long-Lived Credentials Cho Automation

```bash
# Sai — Access key cứng trong code hoặc CI/CD
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

# Đúng — OIDC federation cho GitHub Actions
uses: aws-actions/configure-aws-credentials@v4
with:
  role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole

# Đúng — IAM Role cho EC2/Lambda (không cần key)
# Instance profile tự động lấy credentials từ IMDS
```

---

## 📋 Quick Debug Checklist

Khi gặp "Access Denied", kiểm tra theo thứ tự:

```
□ Principal là ai? (aws sts get-caller-identity)
□ Lỗi là explicit deny hay implicit deny?
□ SCP có block không? (nếu dùng Organizations)
□ Permission Boundary có include action không?
□ Identity policy có allow action không?
□ Resource policy có deny hoặc thiếu principal không?
□ Conditions có fail không? (MFA, IP, time, tags)
□ VPC Endpoint Policy có restrict không? (nếu qua VPC endpoint)
□ Session Policy có giới hạn không? (nếu AssumeRole với policy)
□ Service-linked Role có cần thiết không? (một số dịch vụ cần)
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
