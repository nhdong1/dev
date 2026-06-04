# 5 — Cross-Account Roles (Role Liên Tài Khoản)

> Cross-Account Roles cho phép IAM principals (users, roles, services) trong một AWS account assume (tiếp nhận) IAM Role trong account khác mà không cần tạo IAM user mới — đây là pattern nền tảng cho kiến trúc multi-account trên AWS.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#khái-niệm-cơ-bản)
2. [Cơ Chế Trust Policy](#cơ-chế-trust-policy)
3. [Mô Hình Hub-and-Spoke](#mô-hình-hub-and-spoke)
4. [Cross-Account Role Patterns](#cross-account-role-patterns)
5. [Thực Hành: Thiết Lập Cross-Account Access](#thực-hành-thiết-lập-cross-account-access)
6. [External ID — Bảo Vệ Khỏi Confused Deputy](#external-id--bảo-vệ-khỏi-confused-deputy)
7. [Role Chaining — Chuỗi Role](#role-chaining--chuỗi-role)
8. [Audit Cross-Account Access](#audit-cross-account-access)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cơ Bản

### Tại Sao Cần Cross-Account?

```
Kiến trúc multi-account điển hình của doanh nghiệp:

Management Account (123456789012)
├── Dev Account (234567890123)
├── Staging Account (345678901234)
├── Prod Account (456789012345)
├── Security Account (567890123456)   ← Central logging, GuardDuty
├── Shared Services Account (678901234567)  ← ECR, shared tools
└── Network Account (789012345678)   ← VPC, DNS

Vấn đề: Developer trong "Dev Account" cần:
- Đọc container images từ "Shared Services Account" (ECR)
- Xem security findings từ "Security Account"
- Không có quyền vào "Prod Account" (chỉ CI/CD pipeline mới có)

Giải pháp: Cross-Account IAM Roles
```

### Cách STS AssumeRole Hoạt Động

```
Account A (Trusting = đặt trust)    Account B (Trusted = được trust)
           │                                      │
           │ 1. Role trong Account A trust        │
           │    Principal trong Account B         │
           │                                      │
           │ 2. Principal trong B gọi:            │
           │    sts:AssumeRole                    │
           │    RoleArn: arn:aws:iam::A:role/XYZ  │
           │                                      │
           │ 3. STS verify trust relationship     │
           │                                      │
           │ 4. STS trả về Temporary Credentials  │
           │    (AccessKey + SecretKey + Token)    │
           │                                      │
           │ 5. Principal dùng credentials với    │
           │    permissions của Role XYZ          │
           │                                      │
```

---

## Cơ Chế Trust Policy

### Cấu Trúc Trust Policy

```json
// Role trong Account A (123456789012)
// Trust Policy: "Ai được phép assume role này?"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        // Option 1: Cho phép cả account B
        "AWS": "arn:aws:iam::234567890123:root"
        
        // Option 2: Cho phép specific user trong account B
        // "AWS": "arn:aws:iam::234567890123:user/alice"
        
        // Option 3: Cho phép specific role trong account B
        // "AWS": "arn:aws:iam::234567890123:role/DeployRole"
        
        // Option 4: Cho phép AWS service (Lambda, EC2...)
        // "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        // Chỉ cho phép khi MFA đã bật
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        // Chỉ cho phép từ specific IP range
        "IpAddress": {
          "aws:SourceIp": ["203.x.x.x/24"]
        }
      }
    }
  ]
}
```

### Hai Bên Cần Cho Phép

```
Cross-account access cần CỔNG SỰ CHO PHÉP TỪ CẢ HAI PHÍA:

Phía 1: Trust Policy của Role (Account A cho phép Account B assume)
Phía 2: Permission Policy của caller (Account B cho phép user gọi sts:AssumeRole)

Nếu thiếu một trong hai → Access Denied

Ví dụ:
Account A (123456789012):
  Role "ReadOnlyForB" với trust policy cho phép 234567890123:role/DeveloperRole

Account B (234567890123):
  Role "DeveloperRole" cần có policy:
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::123456789012:role/ReadOnlyForB"
  }

Cả hai phải OK → cross-account access thành công.
```

---

## Mô Hình Hub-and-Spoke

### Kiến Trúc Tổng Thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IDENTITY ACCOUNT (HUB)                            │
│                    (678901234567)                                     │
│                                                                       │
│  IAM Users / IAM Identity Center / SAML Federation                   │
│                                                                       │
│  Role: "Developer" → Cho phép assume spoke roles                     │
│  Role: "DevOps"    → Cho phép assume spoke roles                     │
│  Role: "Auditor"   → Cho phép assume read-only roles in all spokes   │
└──────────────────────┬────────────────────────────────────────────────┘
                       │ AssumeRole
         ┌─────────────┼──────────────┬──────────────────┐
         ▼             ▼              ▼                   ▼
┌────────────┐ ┌────────────┐ ┌────────────┐    ┌────────────┐
│  Dev       │ │  Staging   │ │  Prod      │    │  Security  │
│  Account   │ │  Account   │ │  Account   │    │  Account   │
│ (SPOKE)    │ │ (SPOKE)    │ │ (SPOKE)    │    │ (SPOKE)    │
│            │ │            │ │            │    │            │
│ Role:      │ │ Role:      │ │ Role:      │    │ Role:      │
│ DevAccess  │ │ DevAccess  │ │ ReadOnly   │    │ AuditView  │
│ (trust hub)│ │ (trust hub)│ │ (trust hub)│    │ (trust hub)│
└────────────┘ └────────────┘ └────────────┘    └────────────┘
```

### Trust Policy Của Spoke Roles

```json
// Role "DevAccess" trong Dev Account (trust Hub Account)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        // Trust cả Identity Account
        "AWS": "arn:aws:iam::678901234567:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        // Chỉ cho phép MFA users
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}

// Role "ReadOnly" trong Prod Account (trust chặt hơn, chỉ DevOps role)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        // Chỉ trust specific role trong Hub, không phải toàn bộ hub account
        "AWS": "arn:aws:iam::678901234567:role/DevOps"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

---

## Cross-Account Role Patterns

### Pattern 1: Developer Access Với Role Switching

```bash
# ~/.aws/config — Developer setup
[profile hub]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
region = us-east-1

# Profile để switch sang Dev account
[profile dev]
role_arn = arn:aws:iam::234567890123:role/DevAccess
source_profile = hub
region = us-east-1
mfa_serial = arn:aws:iam::678901234567:mfa/alice  # MFA device ARN

# Profile để switch sang Prod (ReadOnly)
[profile prod-ro]
role_arn = arn:aws:iam::456789012345:role/ReadOnly
source_profile = hub
region = us-east-1
mfa_serial = arn:aws:iam::678901234567:mfa/alice

# Sử dụng:
aws s3 ls --profile dev        # Truy cập Dev account
aws ec2 describe-instances --profile prod-ro  # Xem Prod, read-only
```

### Pattern 2: CI/CD Pipeline Cross-Account Deployment

```yaml
# GitHub Actions: Deploy từ build account sang prod account
name: Deploy to Prod

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Bước 1: Authenticate với Deploy Account (CI/CD account)
      - name: Assume CI Role
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/GitHubCIRole
          aws-region: us-east-1
      
      # Bước 2: Từ CI account, assume role trong Prod account
      - name: Assume Prod Deploy Role
        run: |
          CREDENTIALS=$(aws sts assume-role \
            --role-arn arn:aws:iam::456789012345:role/ProdDeployRole \
            --role-session-name "github-deploy-${GITHUB_RUN_ID}" \
            --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
            --output text)
          
          export AWS_ACCESS_KEY_ID=$(echo $CREDENTIALS | awk '{print $1}')
          export AWS_SECRET_ACCESS_KEY=$(echo $CREDENTIALS | awk '{print $2}')
          export AWS_SESSION_TOKEN=$(echo $CREDENTIALS | awk '{print $3}')
          
          # Deploy với prod credentials
          aws lambda update-function-code \
            --function-name prod-api \
            --zip-file fileb://api.zip
```

### Pattern 3: Lambda Truy Cập Cross-Account

```python
import boto3

def lambda_handler(event, context):
    """
    Lambda trong account A cần đọc DynamoDB trong account B.
    """
    # Lambda's execution role cần sts:AssumeRole permission
    sts = boto3.client('sts')
    
    # Assume role trong account B
    assumed_role = sts.assume_role(
        RoleArn='arn:aws:iam::234567890123:role/DynamoDBReaderRole',
        RoleSessionName='lambda-cross-account-read',
        DurationSeconds=900  # 15 phút — đủ cho một Lambda invocation
    )
    
    credentials = assumed_role['Credentials']
    
    # Tạo DynamoDB client với cross-account credentials
    dynamodb = boto3.resource(
        'dynamodb',
        region_name='us-east-1',
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken']
    )
    
    table = dynamodb.Table('SharedData')
    response = table.get_item(Key={'id': event['id']})
    
    return response.get('Item')
```

```json
// Trust Policy của DynamoDBReaderRole trong Account B
// Chỉ trust Lambda execution role của Account A
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyLambdaExecutionRole"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Pattern 4: Centralized Security Audit (Kiểm Toán Bảo Mật Tập Trung)

```python
import boto3

def audit_all_accounts():
    """
    Từ Security Account, audit tất cả accounts trong Organization.
    """
    accounts = [
        {'id': '234567890123', 'name': 'dev'},
        {'id': '345678901234', 'name': 'staging'},
        {'id': '456789012345', 'name': 'prod'},
    ]
    
    sts = boto3.client('sts')
    findings = []
    
    for account in accounts:
        # Assume audit role trong mỗi account
        assumed = sts.assume_role(
            RoleArn=f"arn:aws:iam::{account['id']}:role/SecurityAuditRole",
            RoleSessionName=f"security-audit-{account['name']}"
        )
        
        creds = assumed['Credentials']
        
        # Kiểm tra S3 buckets có public access không
        s3 = boto3.client(
            's3',
            aws_access_key_id=creds['AccessKeyId'],
            aws_secret_access_key=creds['SecretAccessKey'],
            aws_session_token=creds['SessionToken']
        )
        
        buckets = s3.list_buckets()['Buckets']
        for bucket in buckets:
            try:
                public_access = s3.get_public_access_block(Bucket=bucket['Name'])
                config = public_access['PublicAccessBlockConfiguration']
                
                if not all([
                    config.get('BlockPublicAcls', False),
                    config.get('IgnorePublicAcls', False),
                    config.get('BlockPublicPolicy', False),
                    config.get('RestrictPublicBuckets', False)
                ]):
                    findings.append({
                        'account': account['name'],
                        'account_id': account['id'],
                        'resource': f"s3://{bucket['Name']}",
                        'issue': 'S3 bucket has public access not fully blocked'
                    })
            except Exception as e:
                # Bucket có thể không có public access block config
                findings.append({
                    'account': account['name'],
                    'account_id': account['id'],
                    'resource': f"s3://{bucket['Name']}",
                    'issue': 'Public access block configuration missing'
                })
    
    return findings
```

---

## External ID — Bảo Vệ Khỏi Confused Deputy

### Confused Deputy Attack (Tấn Công Đại Lý Nhầm Lẫn)

```
Tình huống:
- Vendor "AwesomeMonitoring" (account 999999999999) cần assume role trong account bạn
- Vendor phục vụ nhiều khách hàng
- Attacker biết Role ARN của bạn và giả vờ là vendor để lấy credentials của bạn

Không có External ID:
Attacker ──▶ AwesomeMonitoring API: "Lấy data của customer_id=YOUR_ACCOUNT"
AwesomeMonitoring ──▶ STS: AssumeRole RoleArn=arn:aws:iam::YOUR_ACCOUNT:role/MonitoringRole
AWS ──▶ OK (vendor's account được trust!)
Attacker ──▶ Có credentials của bạn thông qua vendor!

Với External ID:
Trust Policy yêu cầu ExternalId="unique-secret-per-customer"
Vendor không biết ExternalId của bạn → không thể assume role bạn
```

### Cấu Hình External ID

```json
// Trust Policy với ExternalId condition
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999999999999:root"  // Vendor account
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          // External ID — bí mật chỉ bạn và vendor biết
          // Vendor phải pass ExternalId khi gọi AssumeRole
          "sts:ExternalId": "my-unique-external-id-abc123xyz"
        }
      }
    }
  ]
}
```

```python
# Vendor code: phải pass ExternalId khi assume role của customer
sts = boto3.client('sts')

response = sts.assume_role(
    RoleArn=f"arn:aws:iam::{customer_account_id}:role/VendorMonitoringRole",
    RoleSessionName='vendor-monitoring',
    ExternalId=customer_external_id  # Mỗi customer có ExternalId riêng
)
```

### Best Practices Cho External ID

```
1. Mỗi khách hàng → External ID khác nhau (UUID hoặc random string)
2. External ID phải đủ entropy (ít nhất 128 bit random)
3. Không dùng: account ID, company name, predictable values
4. Lưu External ID an toàn trong cả hai phía (bạn và vendor)
5. External ID KHÔNG phải secret tuyệt đối — là thêm một lớp bảo vệ

Tạo External ID tốt:
import secrets
external_id = secrets.token_hex(32)  # 64 character hex string
```

---

## Role Chaining — Chuỗi Role

### Role Chaining Là Gì?

```
Role Chaining: Assume role A, dùng credentials của A để assume role B

Account 1                Account 2                Account 3
User ──▶ AssumeRole A ──▶ AssumeRole B (từ A) ──▶ Access Resources

Giới Hạn Quan Trọng:
- Session duration bị giới hạn ở 1 giờ (không thể extend)
- Dù role B cho phép MaxSessionDuration = 12 giờ
- Vì: Chained sessions inherits limitation của parent chain

Ví Dụ Thực Tế:
Developer (hub) → assume DevOps role (hub) → assume ProdDeployRole (prod)
Session của ProdDeployRole bị giới hạn 1 giờ do chaining
```

### Khi Nào Role Chaining Xảy Ra?

```
Xảy ra khi:
1. Bạn đang dùng temporary credentials (đã assume một role)
2. Sau đó assume thêm một role khác

Không phải role chaining khi:
1. IAM user (long-term credentials) → assume role
   (IAM user không bị giới hạn session duration)

Thực tế:
- EC2 Instance Profile → AssumeRole cross-account = Role Chaining (1h limit)
- Lambda Execution Role → AssumeRole cross-account = Role Chaining (1h limit)
- GitHub Actions OIDC role → AssumeRole cross-account = Role Chaining (1h limit)

1 giờ thường là đủ cho CI/CD pipeline và automated tasks.
Nếu cần hơn 1 giờ, cần thiết kế lại flow.
```

---

## Thực Hành: Thiết Lập Cross-Account Access

### Thiết Lập Hoàn Chỉnh Với Terraform

```hcl
# ========= ACCOUNT B (Trust Account / Spoke) =========
# File: accounts/prod/cross_account_roles.tf

# Role mà Account A sẽ assume
resource "aws_iam_role" "readonly_from_hub" {
  name = "ReadOnlyFromHub"
  
  # Trust Policy: ai được phép assume role này
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          # Chỉ trust specific roles trong hub, không phải cả hub account
          AWS = [
            "arn:aws:iam::${var.hub_account_id}:role/DeveloperRole",
            "arn:aws:iam::${var.hub_account_id}:role/AuditorRole"
          ]
        }
        Action = "sts:AssumeRole"
        Condition = {
          Bool = {
            "aws:MultiFactorAuthPresent" = "true"
          }
        }
      }
    ]
  })
  
  tags = {
    Purpose = "Cross-account read-only access from hub"
    TrustedAccount = var.hub_account_id
  }
}

# Gắn policy cho role
resource "aws_iam_role_policy_attachment" "readonly_attach" {
  role       = aws_iam_role.readonly_from_hub.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}


# ========= ACCOUNT A (Hub Account) =========
# File: accounts/hub/assume_role_permissions.tf

# Policy cho phép developer assume role trong prod
resource "aws_iam_policy" "assume_prod_readonly" {
  name = "AssumeProdReadOnly"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "sts:AssumeRole"
        Resource = "arn:aws:iam::${var.prod_account_id}:role/ReadOnlyFromHub"
      }
    ]
  })
}

# Gắn vào Developer group
resource "aws_iam_group_policy_attachment" "developer_assume_prod" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.assume_prod_readonly.arn
}
```

### Kiểm Tra Cross-Account Access

```bash
# Từ Account A (Hub), verify có thể assume role trong Account B (Prod)
aws sts assume-role \
  --role-arn arn:aws:iam::456789012345:role/ReadOnlyFromHub \
  --role-session-name test-cross-account \
  --serial-number arn:aws:iam::678901234567:mfa/alice \
  --token-code 123456

# Verify caller identity sau khi assume
aws sts get-caller-identity

# Verify access
aws s3 ls --region us-east-1
```

---

## Audit Cross-Account Access

### CloudTrail Events

```json
// AssumeRole event trong CloudTrail của Account B (role được assume)
{
  "eventName": "AssumeRole",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXXXXXXXXXX:alice",
    "arn": "arn:aws:sts::678901234567:assumed-role/DeveloperRole/alice",
    "accountId": "678901234567"  // Account A gọi AssumeRole
  },
  "requestParameters": {
    "roleArn": "arn:aws:iam::456789012345:role/ReadOnlyFromHub",
    "roleSessionName": "alice-prod-readonly"
  },
  "responseElements": {
    "assumedRoleUser": {
      "assumedRoleId": "AROAXXXXXXXXXXXXXXXXX:alice-prod-readonly",
      "arn": "arn:aws:sts::456789012345:assumed-role/ReadOnlyFromHub/alice-prod-readonly"
    },
    "credentials": {
      "sessionToken": "<token>",
      "accessKeyId": "ASIAXXXXXXXXXXXXXXXXX",
      "expiration": "2026-05-16T10:00:00Z"
    }
  },
  "sourceIPAddress": "203.x.x.x"
}
```

### Query Phân Tích Cross-Account Access

```sql
-- CloudTrail Lake query: Tất cả cross-account AssumeRole trong 7 ngày qua
SELECT
    eventtime,
    useridentity.accountid as source_account,
    useridentity.arn as source_principal,
    requestparameters.rolearn as assumed_role,
    sourceipaddress,
    awsregion
FROM <cloudtrail-lake-event-data-store-arn>
WHERE eventname = 'AssumeRole'
  AND useridentity.accountid != '456789012345'  -- Chỉ cross-account events
  AND eventtime > ago(7d)
ORDER BY eventtime DESC;
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích cơ chế Cross-Account Role từ đầu đến cuối

```
Ví dụ: Developer trong Hub Account (111) cần truy cập S3 trong Prod Account (222)

Setup (làm một lần):
1. Trong Prod Account (222): Tạo role "ProdReadOnly"
   - Trust Policy: trust "arn:aws:iam::111:role/DeveloperRole"
   - Permission Policy: s3:GetObject, s3:ListBucket
   
2. Trong Hub Account (111): Role "DeveloperRole"
   - Permission Policy: sts:AssumeRole trên "arn:aws:iam::222:role/ProdReadOnly"

Runtime (mỗi khi developer cần truy cập):
1. Developer đang dùng Hub Account session (via DeveloperRole)
2. Gọi: aws sts assume-role --role-arn arn:aws:iam::222:role/ProdReadOnly
3. AWS STS kiểm tra:
   - Caller có permission sts:AssumeRole trên role đó không? ✅
   - Trust policy của ProdReadOnly có trust caller không? ✅
4. STS trả về Temporary Credentials (expire 1 giờ)
5. Developer dùng credentials đó truy cập S3 trong Prod Account

Audit: CloudTrail trong Prod Account ghi lại AssumeRole event với đầy đủ context
```

### Q2: External ID là gì và khi nào cần dùng?

```
External ID là thêm một lớp xác minh trong trust relationship với bên thứ ba.

Cần dùng khi:
- Một vendor/third-party service cần assume role trong account bạn
- Vendor phục vụ nhiều customers (confused deputy attack risk)

Không cần khi:
- Cross-account trong cùng Organization (bạn kiểm soát cả hai phía)
- Service-to-service trong accounts bạn own

Cách hoạt động:
1. Bạn generate External ID (UUID) và chia sẻ với vendor
2. Trust policy của role require ExternalId condition
3. Khi vendor gọi AssumeRole, phải pass ExternalId đúng
4. Nếu attacker không biết External ID → không thể assume role

Lưu ý: External ID KHÔNG phải secret theo nghĩa tuyệt đối —
nó ngăn attack theo kiểu vendor giả vờ là customer,
không ngăn được nếu External ID bị lộ.
```

### Q3: Tại sao Role Chaining giới hạn session 1 giờ?

```
Role Chaining: Dùng temporary credentials để assume role khác.

AWS giới hạn 1 giờ vì:
- Security: Giới hạn exposure window của chained sessions
- Nếu không giới hạn: User có thể chain roles liên tục và giữ elevated access mãi mãi
- Temporary credentials → temporary, không phải semi-permanent

Giải pháp khi cần hơn 1 giờ:
1. Sử dụng IAM user (long-term) → AssumeRole (không phải chaining)
   Nhưng: IAM users không phải best practice
   
2. Re-design workflow để không cần session dài:
   - Chia task thành các steps ngắn hơn 1 giờ
   - Mỗi step re-assume role khi cần
   
3. Dùng IAM Identity Center (không bị giới hạn role chaining 1 giờ
   khi user là human, session up to 12 giờ)

Trong practice: CI/CD pipelines thường chạy trong < 30 phút,
nên 1 giờ limit không phải vấn đề thực tế.
```

### Q4: Làm thế nào thiết kế cross-account access cho 50 accounts một cách scalable?

```
Giải pháp scalable: Standardized Role Names + Centralized Permission Sets

1. Chuẩn hóa role names trong mỗi account:
   - "DevReadOnly" — developer read-only access
   - "DevAccess" — developer full dev environment access
   - "OpsAccess" — operations team full access
   - "SecurityAudit" — security team read-only everywhere
   - "EmergencyAdmin" — break-glass full admin

2. Dùng IAM Identity Center (khuyến nghị mạnh):
   - Define Permission Sets một lần
   - Gán vào accounts thông qua Organizations API
   - Identity Center tự động tạo/maintain roles trong mỗi account
   - Scale dễ dàng khi thêm accounts mới

3. Nếu không dùng Identity Center:
   - Dùng CloudFormation StackSets để deploy roles vào tất cả accounts
   - Centralize management trong management account
   - Tạo template chuẩn cho "spoke account roles"

4. Governance (Quản Trị):
   - SCPs (Service Control Policies) trong Organizations ngăn xóa audit roles
   - AWS Config rule detect roles không conform với standard
   - Regular access reviews (kiểm tra định kỳ ai có quyền gì)
```

---

## 🔗 Điều Hướng

- **Trước:** [4-cognito-user-pools.md](4-cognito-user-pools.md) — Amazon Cognito
- **Module tiếp theo:** [../03-organizations/README.md](../03-organizations/README.md) — AWS Organizations
- **Liên quan:** [../01-iam-fundamentals/2-policy-types.md](../01-iam-fundamentals/2-policy-types.md) — Policy Types

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
